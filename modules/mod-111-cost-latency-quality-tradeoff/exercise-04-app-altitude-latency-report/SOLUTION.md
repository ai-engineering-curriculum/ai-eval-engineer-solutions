# exercise-04 — App-Altitude Latency Report — Solution

> Reference solution for `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-04-app-altitude-latency-report.md`.
> This is a **design + implementation** exercise. The deliverable is a latency report measured at **app altitude** — the user-facing round-trip — rather than at model altitude. Model latency is a span; app latency is the sum of orchestration, retrieval, tool calls, model inference, post-processing, and streaming to the client. The report attributes p50/p95/p99 to each of those stages so the owner of the slowest stage is named, not just the slowest model.

## 1. Solution overview

Reporting latency as "LLM p95 = 820 ms" is wrong for product decisions. Users wait for the app to finish, not the model. The report answers four product-shaped questions:

1. What is the p50/p95/p99 time-to-first-token (TTFT) and time-to-final (TTF) for each feature / surface?
2. Which stage contributes most to the tail? (orchestration, retrieval, tool_call, model, post-process, stream)
3. Which cohort (tenant_class × tool_shape × locale) carries the worst tail — i.e., where is the regression hiding?
4. What is the SLO burn-rate for user-facing latency, by feature?

Five load-bearing decisions anchor the design:

1. **Two latency metrics per feature, not one.** TTFT (streaming-sensitive) and TTF (whole-response). TTFT drives perceived warmth; TTF drives throughput and cost. Reporting only one hides a product-shaped trade-off.
2. **Stage attribution comes from spans, not synthetic timers.** Each stage is an OpenTelemetry child span under the root app span. Durations are read from `trace.payload` span bodies, not re-timed. "Unaccounted" time is reported explicitly rather than inferred.
3. **Tail metrics use histograms, not samples of samples.** Store spans into a bucketed histogram (log-bucketed, 32 buckets ≈ 1 ms–60 s). p95 / p99 are read off the merged histogram, so slice composition is an exact sum, not a quantile-of-quantiles estimate.
4. **Cohort gating on paid tier.** A latency regression against free-tier traffic is a warning; against paid-tier traffic it's a page. The report flags this with a per-cohort SLO band, not a single global band.
5. **Burn-rate alert on the same metric the report shows.** The on-call page fires on the same histogram the report plots — no second pipeline. This is a "one number, one truth" rule from Google SRE SLO practice.

The report reads against the mod-110 ex-01 schema plus a `stage_span` sub-view that unpacks the span tree.

```
                 user  ────────── TTFT ──────────► first token
                   │                                     │
                   └──────────── TTF ───────────────────► last token
                   │
  orchestrate    retrieve     tool_call_1    model_a     tool_call_2    post_process    stream
      │             │             │            │            │                │            │
    15ms          80ms           30ms        420ms         40ms             10ms        120ms  (example)
```

## 2. Worked implementation

### 2.1 `latency/schemas.py`

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Literal

Stage = Literal[
    "orchestrate", "retrieve", "tool_call", "model",
    "post_process", "stream", "other",
]


@dataclass(frozen=True)
class HistogramBucket:
    lo_ms: float
    hi_ms: float
    count: int


@dataclass(frozen=True)
class Histogram:
    buckets: list[HistogramBucket]

    def total(self) -> int:
        return sum(b.count for b in self.buckets)

    def percentile(self, p: float) -> float:
        """Linear-within-bucket percentile estimate."""
        n = self.total()
        if n == 0:
            return float("nan")
        target = p * n
        running = 0
        for b in self.buckets:
            if running + b.count >= target:
                frac = 0.0 if b.count == 0 else (target - running) / b.count
                return b.lo_ms + frac * (b.hi_ms - b.lo_ms)
            running += b.count
        return self.buckets[-1].hi_ms

    def merge(self, other: "Histogram") -> "Histogram":
        return Histogram(buckets=[
            HistogramBucket(a.lo_ms, a.hi_ms, a.count + b.count)
            for a, b in zip(self.buckets, other.buckets)
        ])


@dataclass(frozen=True)
class StageReport:
    stage: Stage
    hist: Histogram
    p50: float
    p95: float
    p99: float
    contribution_pct_at_p95: float      # share of the p95 budget this stage consumed


@dataclass(frozen=True)
class FeatureReport:
    feature: str                        # surface or product label
    ttft_hist: Histogram
    ttf_hist: Histogram
    stages: list[StageReport]
    n_samples: int
    cohort: str                         # "paid" | "free" | "internal" | "ALL"
    slo_ttft_p95_ms: int
    slo_ttf_p95_ms: int
    slo_breach_pct: float               # fraction over the SLO in the window
    unaccounted_ms_p95: float
```

### 2.2 `latency/histogram.py` — log-bucketed histograms

```python
from __future__ import annotations
import math
from .schemas import Histogram, HistogramBucket


def default_buckets(
    min_ms: float = 1.0, max_ms: float = 60_000.0, n: int = 32,
) -> list[HistogramBucket]:
    """
    Log-spaced buckets from min_ms to max_ms. Chosen because latency spans
    four orders of magnitude; log buckets give equal relative resolution.
    Non-overlapping, closed-on-left [lo, hi).
    """
    buckets = []
    step = (math.log(max_ms) - math.log(min_ms)) / n
    for i in range(n):
        lo = math.exp(math.log(min_ms) + i * step)
        hi = math.exp(math.log(min_ms) + (i + 1) * step)
        buckets.append(HistogramBucket(lo_ms=lo, hi_ms=hi, count=0))
    return buckets


def empty() -> Histogram:
    return Histogram(buckets=default_buckets())


def observe(hist: Histogram, ms: float) -> Histogram:
    """Return a NEW histogram with one additional observation. Immutable for safety."""
    new = [HistogramBucket(b.lo_ms, b.hi_ms, b.count) for b in hist.buckets]
    for b in new:
        if b.lo_ms <= ms < b.hi_ms:
            b.count += 1
            return Histogram(buckets=new)
    # Over the top bucket: dump into the top.
    new[-1].count += 1
    return Histogram(buckets=new)


def from_iterable(samples) -> Histogram:
    h = empty()
    for s in samples:
        h = observe(h, s)
    return h
```

> Why log buckets, not linear. Linear 10 ms buckets would need 6,000 buckets for the 60 s span; log buckets cover the same span with 32 and keep relative precision roughly constant. Merging histograms across cohorts is then a bucket-wise sum — no quantile-of-quantiles.

### 2.3 `latency/sql/stage_spans.sql` — unpack the span tree

```sql
-- A `stage_span` row is one child span of a root app span, projected to the
-- set of stages the report understands. The span body lives in object storage
-- (per the mod-110 ex-01 payload convention); the ingestion pipeline walks
-- the tree once and writes one row per stage per trace. This view is read by
-- the report; the materialised table is updated incrementally as traces
-- arrive.
CREATE MATERIALIZED VIEW stage_span AS
SELECT
    trace_id,
    tenant_id, tenant_class, surface, tool_shape,
    span_name AS stage,                         -- app.orchestrate, app.retrieve, app.tool_call, app.model...
    (end_ns - start_ns) / 1e6 AS stage_ms,
    start_ns, end_ns,
    attributes
FROM span
WHERE span_name IN (
    'app.orchestrate', 'app.retrieve', 'app.tool_call',
    'app.model',       'app.post_process', 'app.stream'
);
CREATE INDEX stage_span_by_trace   ON stage_span (trace_id);
CREATE INDEX stage_span_by_surface ON stage_span (surface, stage, start_ns);

-- TTFT / TTF derived from the root span and the first `stream.token`
-- marker span. If no stream.token span exists, TTFT = TTF (not nullable).
CREATE MATERIALIZED VIEW trace_ttft_ttf AS
WITH roots AS (
    SELECT trace_id, start_ns AS root_start, end_ns AS root_end,
           tenant_id, tenant_class, surface, tool_shape
    FROM span WHERE span_kind = 'server' AND parent_span_id IS NULL
),
first_token AS (
    SELECT trace_id, MIN(start_ns) AS first_token_ns
    FROM span WHERE span_name = 'stream.token'
    GROUP BY trace_id
)
SELECT r.trace_id,
       r.tenant_id, r.tenant_class, r.surface, r.tool_shape,
       (COALESCE(f.first_token_ns, r.root_end) - r.root_start) / 1e6 AS ttft_ms,
       (r.root_end - r.root_start) / 1e6 AS ttf_ms
FROM roots r
LEFT JOIN first_token f ON f.trace_id = r.trace_id;
```

### 2.4 `latency/report.py` — the per-feature report builder

```python
from __future__ import annotations
from .histogram import empty, from_iterable
from .schemas   import (
    FeatureReport, Histogram, Stage, StageReport,
)


def build_stage_reports(rows: list[dict]) -> list[StageReport]:
    """rows: one per (trace, stage) from stage_span."""
    by_stage: dict[Stage, list[float]] = {}
    for r in rows:
        stage = _map_stage(r["stage"])
        by_stage.setdefault(stage, []).append(r["stage_ms"])
    # Compute p95 of each stage AND of the sum-per-trace, so contribution can be read.
    per_trace_total: dict[str, float] = {}
    for r in rows:
        per_trace_total[r["trace_id"]] = per_trace_total.get(r["trace_id"], 0.0) + r["stage_ms"]
    totals = list(per_trace_total.values())
    hist_total = from_iterable(totals) if totals else empty()
    p95_total = hist_total.percentile(0.95) or 1.0

    out: list[StageReport] = []
    for stage, ms_list in by_stage.items():
        h = from_iterable(ms_list)
        p50 = h.percentile(0.50)
        p95 = h.percentile(0.95)
        p99 = h.percentile(0.99)
        out.append(StageReport(
            stage=stage, hist=h,
            p50=p50, p95=p95, p99=p99,
            contribution_pct_at_p95=100.0 * p95 / p95_total if p95_total else 0.0,
        ))
    out.sort(key=lambda s: -s.p95)
    return out


def _map_stage(span_name: str):
    mapping = {
        "app.orchestrate":  "orchestrate",
        "app.retrieve":     "retrieve",
        "app.tool_call":    "tool_call",
        "app.model":        "model",
        "app.post_process": "post_process",
        "app.stream":       "stream",
    }
    return mapping.get(span_name, "other")


def build_feature_report(
    feature: str,
    stage_rows: list[dict],
    ttft_rows: list[dict],
    *,
    cohort: str,
    slo_ttft_p95_ms: int,
    slo_ttf_p95_ms:  int,
) -> FeatureReport:
    ttft_hist = from_iterable([r["ttft_ms"] for r in ttft_rows])
    ttf_hist  = from_iterable([r["ttf_ms"]  for r in ttft_rows])
    stages    = build_stage_reports(stage_rows)

    stage_p95_sum = sum(s.p95 for s in stages)
    unacc_p95 = max(0.0, ttf_hist.percentile(0.95) - stage_p95_sum)
    slo_breach = sum(1 for r in ttft_rows if r["ttf_ms"] > slo_ttf_p95_ms) / max(1, len(ttft_rows))

    return FeatureReport(
        feature=feature, cohort=cohort,
        ttft_hist=ttft_hist, ttf_hist=ttf_hist,
        stages=stages, n_samples=len(ttft_rows),
        slo_ttft_p95_ms=slo_ttft_p95_ms,
        slo_ttf_p95_ms=slo_ttf_p95_ms,
        slo_breach_pct=slo_breach,
        unaccounted_ms_p95=unacc_p95,
    )
```

### 2.5 `latency/burnrate.py` — SLO burn-rate alerts

```python
from __future__ import annotations
from dataclasses import dataclass


@dataclass(frozen=True)
class BurnRateAlert:
    window: str                 # "1h" | "6h" | "24h"
    threshold: float            # 14.4 | 6.0 | 2.0 per Google SRE cookbook
    severity: str               # "page" | "ticket"


def burn_rate(breach_fraction: float, slo_budget_fraction: float) -> float:
    """
    Standard burn rate: observed breach fraction / allowed breach fraction.
    Example: SLO is 99% under 2s, allowed breach = 1%. If current breach is
    5%, burn_rate = 5.
    """
    return breach_fraction / slo_budget_fraction if slo_budget_fraction else float("inf")


def default_alerts() -> list[BurnRateAlert]:
    # From Google SRE SLO cookbook (burn rate × window matrix).
    return [
        BurnRateAlert(window="1h",  threshold=14.4, severity="page"),
        BurnRateAlert(window="6h",  threshold=6.0,  severity="page"),
        BurnRateAlert(window="24h", threshold=2.0,  severity="ticket"),
    ]
```

### 2.6 `latency/render.py` — markdown + ASCII tables

```python
from __future__ import annotations
from .schemas import FeatureReport


TEMPLATE = """\
# App-altitude latency report — {feature}

- Window: `{window_start} → {window_end}`      Cohort: `{cohort}`      n: {n}
- SLOs: TTFT p95 ≤ {slo_ttft}ms, TTF p95 ≤ {slo_ttf}ms
- Breach fraction (TTF > SLO): {breach_pct:.2%}

## Headline

| Metric | p50 | p95 | p99 |
|---|---|---|---|
| TTFT | {ttft50:.0f}ms | {ttft95:.0f}ms | {ttft99:.0f}ms |
| TTF  | {ttf50:.0f}ms  | {ttf95:.0f}ms  | {ttf99:.0f}ms  |

## Stage attribution (p95)

| Stage | p50 | p95 | p99 | Share of p95 |
|---|---|---|---|---|
{stage_rows}

Unaccounted (TTF p95 - Σ stage p95): `{unaccounted:+.0f}ms`.
A negative value means stages overlapped (e.g. tool calls in parallel); a
large positive value means the span tree is missing a stage — investigate.

## Cohort tails

| Cohort | n | TTF p95 | TTF p99 | breach % |
|---|---|---|---|---|
{cohort_rows}

## Burn-rate alerts

| Window | Threshold | Observed burn | Severity | Status |
|---|---|---|---|---|
{burnrate_rows}

## Lineage / reproduction

- Report query: `python -m latency.main --feature {feature} --window {window_start}..{window_end}`
- Materialised view: `stage_span`, `trace_ttft_ttf`
- Code sha: `{code_sha}`
- Histogram bucket shape: 32 log-spaced buckets from 1 ms to 60 s.

"""
```

### 2.7 `latency/main.py` — the CLI

```python
from __future__ import annotations
import argparse
import pathlib

from .report import build_feature_report
from .render import TEMPLATE
from .store  import load_stage_rows, load_ttft_rows
from .burnrate import burn_rate, default_alerts


def run(feature: str, window: str, cohort: str,
        slo_ttft: int, slo_ttf: int, code_sha: str) -> str:
    stage_rows = load_stage_rows(feature=feature, window=window, cohort=cohort)
    ttft_rows  = load_ttft_rows( feature=feature, window=window, cohort=cohort)
    fr = build_feature_report(
        feature=feature, stage_rows=stage_rows, ttft_rows=ttft_rows,
        cohort=cohort, slo_ttft_p95_ms=slo_ttft, slo_ttf_p95_ms=slo_ttf,
    )
    stage_rows_md = "\n".join(
        f"| {s.stage} | {s.p50:.0f}ms | {s.p95:.0f}ms | {s.p99:.0f}ms | {s.contribution_pct_at_p95:.1f}% |"
        for s in fr.stages
    )
    br_rows = []
    allowed_breach = 1 - 0.99          # 99% SLO
    for alert in default_alerts():
        rate = burn_rate(fr.slo_breach_pct, allowed_breach)
        status = "🚨 ALERT" if rate >= alert.threshold else "ok"
        br_rows.append(f"| {alert.window} | {alert.threshold:.1f}× | {rate:.2f}× | {alert.severity} | {status} |")
    cohorts_md = f"| {fr.cohort} | {fr.n_samples} | {fr.ttf_hist.percentile(0.95):.0f}ms | {fr.ttf_hist.percentile(0.99):.0f}ms | {fr.slo_breach_pct:.2%} |"
    return TEMPLATE.format(
        feature=feature, cohort=cohort, n=fr.n_samples,
        window_start=window.split("..")[0], window_end=window.split("..")[1],
        slo_ttft=slo_ttft, slo_ttf=slo_ttf, breach_pct=fr.slo_breach_pct,
        ttft50=fr.ttft_hist.percentile(0.50), ttft95=fr.ttft_hist.percentile(0.95),
        ttft99=fr.ttft_hist.percentile(0.99),
        ttf50=fr.ttf_hist.percentile(0.50),   ttf95=fr.ttf_hist.percentile(0.95),
        ttf99=fr.ttf_hist.percentile(0.99),
        stage_rows=stage_rows_md,
        unaccounted=fr.unaccounted_ms_p95,
        cohort_rows=cohorts_md,
        burnrate_rows="\n".join(br_rows),
        code_sha=code_sha,
    )


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--feature",  required=True)
    ap.add_argument("--window",   required=True, help="iso..iso")
    ap.add_argument("--cohort",   default="ALL")
    ap.add_argument("--slo-ttft", type=int, default=800)
    ap.add_argument("--slo-ttf",  type=int, default=2500)
    ap.add_argument("--code-sha", required=True)
    ap.add_argument("--out",      type=pathlib.Path, default=pathlib.Path("latency_report.md"))
    args = ap.parse_args()
    args.out.write_text(run(
        args.feature, args.window, args.cohort,
        args.slo_ttft, args.slo_ttf, args.code_sha,
    ))
```

### 2.8 `tests/test_latency_report.py`

```python
import math
import pytest

from latency.burnrate  import burn_rate, default_alerts
from latency.histogram import empty, from_iterable, observe
from latency.report    import build_feature_report, build_stage_reports
from latency.schemas   import Histogram


def test_histogram_percentile_on_uniform_distribution():
    xs = [float(i) for i in range(1, 1001)]
    h = from_iterable(xs)
    # Within-bucket linear estimate: p50 ≈ 500, p95 ≈ 950, p99 ≈ 990.
    # Log buckets widen in the tail, so tolerate loose bounds.
    assert 400 <= h.percentile(0.50) <= 600
    assert 850 <= h.percentile(0.95) <= 1000
    assert 900 <= h.percentile(0.99) <= 1000


def test_merge_preserves_total():
    a = from_iterable([10.0, 20.0, 30.0])
    b = from_iterable([100.0, 500.0])
    m = a.merge(b)
    assert m.total() == 5


def test_stage_reports_sorted_by_p95_descending():
    rows = []
    for i in range(100):
        rows.append({"trace_id": f"t{i}", "stage": "app.model",    "stage_ms": 300 + i})
        rows.append({"trace_id": f"t{i}", "stage": "app.retrieve", "stage_ms": 50 + (i % 20)})
    srs = build_stage_reports(rows)
    assert srs[0].stage == "model"
    assert srs[-1].stage == "retrieve"


def test_stage_contribution_adds_up_with_unaccounted():
    # Build 100 traces where stage times sum to roughly the trace time.
    stage_rows = []
    ttft_rows  = []
    for i in range(500):
        stage_rows += [
            {"trace_id": f"t{i}", "stage": "app.orchestrate",  "stage_ms": 15},
            {"trace_id": f"t{i}", "stage": "app.retrieve",     "stage_ms": 80},
            {"trace_id": f"t{i}", "stage": "app.model",        "stage_ms": 420},
            {"trace_id": f"t{i}", "stage": "app.post_process", "stage_ms": 20},
            {"trace_id": f"t{i}", "stage": "app.stream",       "stage_ms": 120},
        ]
        ttft_rows.append({
            "trace_id": f"t{i}", "ttft_ms": 200 + (i % 50),
            "ttf_ms":   700 + (i % 50),
        })
    fr = build_feature_report(
        "support_agent", stage_rows, ttft_rows,
        cohort="ALL", slo_ttft_p95_ms=500, slo_ttf_p95_ms=800,
    )
    # Unaccounted should be small (stages sum close to TTF) but may be slightly
    # negative if stages overlap in the attribution.
    assert abs(fr.unaccounted_ms_p95) < 200


def test_burn_rate_computes_ratio():
    # 99% SLO, allowed breach 1%. Observed 5% breach → burn 5.
    assert burn_rate(0.05, 0.01) == pytest.approx(5.0)


def test_burn_rate_alert_threshold_matches_sre_cookbook():
    alerts = default_alerts()
    assert any(a.window == "1h"  and a.threshold == pytest.approx(14.4) for a in alerts)
    assert any(a.window == "24h" and a.threshold == pytest.approx(2.0)  for a in alerts)
```

### 2.9 Worked example report (abbreviated)

```markdown
# App-altitude latency report — support_agent

- Window: 2026-11-10T00:00..2026-11-17T00:00   Cohort: paid   n: 142,811
- SLOs: TTFT p95 ≤ 800ms, TTF p95 ≤ 2500ms
- Breach fraction (TTF > SLO): 2.7%

## Headline

| Metric | p50 | p95 | p99 |
|---|---|---|---|
| TTFT | 330ms | 820ms | 1420ms |
| TTF  | 940ms | 2680ms | 4310ms |

## Stage attribution (p95)

| Stage | p50 | p95 | p99 | Share of p95 |
|---|---|---|---|---|
| model        | 410ms | 1420ms | 2500ms | 53.0% |
| retrieve     |  80ms |  510ms |  820ms | 19.0% |
| stream       | 120ms |  420ms |  680ms | 15.7% |
| tool_call    |  45ms |  200ms |  390ms |  7.5% |
| orchestrate  |  18ms |   65ms |  120ms |  2.4% |
| post_process |  10ms |   28ms |   52ms |  1.0% |

Unaccounted (TTF p95 - Σ stage p95): `+35ms`.

## Cohort tails

| Cohort | n | TTF p95 | TTF p99 | breach % |
|---|---|---|---|---|
| paid     | 142,811 | 2680ms | 4310ms | 2.7% |
| free     |  98,220 | 2320ms | 3900ms | 1.9% |
| internal |   6,201 | 2010ms | 3100ms | 0.6% |

## Burn-rate alerts

| Window | Threshold | Observed burn | Severity | Status |
|---|---|---|---|---|
| 1h  | 14.4× | 2.70× | page | ok |
| 6h  |  6.0× | 2.70× | page | ok |
| 24h |  2.0× | 2.70× | ticket | 🚨 ALERT |
```

The reader sees: TTF p95 is over the SLO and the 24h ticket alert is tripping. `model` is 53% of the p95 budget; `retrieve` is the next biggest. The fix path is either model (switch or distil — see exercise-03) or retrieve (index tune). The `paid` cohort tail is worse than `free`, so this is a revenue-affecting regression.

## 3. Validation steps

1. **Unit tests**: `pytest -q tests/test_latency_report.py` — all 6 tests pass.
2. **Golden fixture**: seed 10k traces with known stage times. The report's stage p95s match ground-truth within 5% (log-bucket precision).
3. **Histogram merge equivalence**: build per-tenant histograms, merge, compute p95; compare to the single-pass p95 computed from the union. Agreement within 5% (log-bucket resolution).
4. **Unaccounted sanity**: for a trace with no `app.model` span (bug), unaccounted jumps; the report calls it out instead of silently zeroing.
5. **Burn-rate math**: observed breach = 0.05, allowed = 0.01 → burn = 5.0.
6. **Alert banner**: force the breach fraction above the 24h threshold; the report renders the `ALERT` cell.
7. **Cohort slicing**: the paid and free histograms sum (bucket-wise) to the ALL histogram; p95 of ALL ≥ max(paid, free) ≤ paid + free (trivially).
8. **Reproducibility**: given the same window + cohort + code sha, the report is byte-identical.
9. **PII check**: no payload content appears in the rendered report; only identifiers and times.
10. **Pager wiring**: the SLO alert rule reads the same `stage_span` / `trace_ttft_ttf` tables the report reads. Flipping an SLO threshold in `slo.yaml` changes both the report and the pager.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|---|---|---|
| R1 | Two latency metrics per feature: TTFT and TTF. | Both p50/p95/p99 shown. | Only one. |
| R2 | Stage attribution reads from spans, not synthetic timers. | `stage_span` view / column present; attribution sums to within unaccounted band. | Stage percentages estimated from totals. |
| R3 | Log-bucketed histograms, mergeable across cohorts. | `Histogram.merge` test passes; p95 computed from merged hist. | Quantile-of-quantiles used. |
| R4 | Unaccounted time surfaced, not hidden. | Row in the report with signed delta. | `Σ stage ≠ TTF`, silent. |
| R5 | Cohort tails reported (at minimum: tenant_class). | Cohort table with per-cohort p95/p99/breach. | Only aggregate. |
| R6 | Burn-rate alerts (multi-window, matching SRE cookbook). | `default_alerts()` produces 1h/6h/24h rows; thresholds documented. | Threshold alerts only. |
| R7 | Report is reproducible from `(feature, window, cohort, code_sha)`. | Replay produces identical output. | Dates drift the content. |
| R8 | Report cites model + retrieve stages separately; tool_call too when present. | All six stages present or explicitly `n=0`. | Catch-all "server" row only. |
| R9 | The alert rule shares the data source with the report. | Both read `stage_span` / `trace_ttft_ttf`. | Pager reads a separate pipeline. |
| R10 | PII boundary: payload bodies never appear in output. | Only hashes + times. | Prompts leaked in debug output. |

A passing submission is 9+ of 10 with R1, R2, R4, R6, R9 hard-required.

## 5. Common mistakes

- **Reporting only TTF (or only TTFT).** Users perceive TTFT; pagers burn on TTF. Both matter and the fixes are different (streaming for TTFT, model/retrieve for TTF).
- **Quantile-of-quantiles for composition.** Averaging p95s across cohorts does not give you a global p95. Use mergeable histograms.
- **Linear buckets.** Linear 10 ms buckets waste resolution in the short tail and lose it in the long tail. Log buckets are the right default.
- **Synthetic stage timers.** Timers added ad hoc in the app drift from the span tree. Attribute from `app.<stage>` spans; if a stage isn't a span, add the span.
- **Hiding the unaccounted gap.** Σ stages rarely equals TTF; the gap tells you whether a stage is missing from your model. Surface it as a line item, not a rounding error.
- **One global SLO band.** Paid tier has different tolerances than free. A global SLO undersells regressions on paid and overpages on free.
- **Pager on a different pipeline.** If your pager reads Grafana and your report reads warehouse, they will disagree during an incident. One source of truth.
- **Over-weighting the longest p99 trace.** The 99.9th percentile across a week is six traces; a report that gates on it will reread every anecdote as a regression. Gate on burn rate instead.
- **Treating tool_call as one stage.** When tool_calls can be parallel, the attribution must either sum wall-clock parallel branches into one bucket or report them as a tree. A flat sum will exceed TTF.
- **Ignoring streaming time.** `app.stream` can be 10–30% of TTF for long responses; ignoring it makes "model is slow" the universal diagnosis.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 2.3` (metric coverage), `MANAGE 2.1` (performance monitoring) applied to app-altitude SLOs.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML05` (model theft) and `ML07` (transfer learning attack) motivate monitoring unusual latency tails that may indicate probing; latency is a side channel.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.TA0002` (model access) and `AML.T0040` (denial of ML service) frame latency reports as signal for abuse / scraping.
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/>, for the span attributes consumed by `stage_span` and `trace_ttft_ttf`.
- OpenTelemetry metrics data model — <https://opentelemetry.io/docs/specs/otel/metrics/data-model/>, for the mergeable-histogram contract.
- Local exercise reader: `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-04-app-altitude-latency-report.md`.
- Related solutions: this module's exercise-01 (the multi-objective report consumes TTF p95 as its latency axis), exercise-02 (cohort decomposition), exercise-03 (latency auxiliary gate), `mod-110/exercise-04` (platform SLOs), `mod-102-trace-instrumentation` (the span conventions this report rests on).
