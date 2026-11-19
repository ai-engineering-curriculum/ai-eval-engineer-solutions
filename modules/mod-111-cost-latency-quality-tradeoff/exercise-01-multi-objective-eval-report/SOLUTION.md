# exercise-01 — Multi-Objective Eval Report — Solution

> Reference solution for `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-01-multi-objective-eval-report.md`.
> This is a **design + implementation** exercise. The deliverable is a repeatable report that compares two or more candidate configurations (model, prompt, or pipeline) along three orthogonal axes — **quality**, **cost**, and **latency** — in a way that makes the trade-off decidable rather than vibes-based. The report pairs a Pareto view (so a reader sees which candidates are dominated) with a weighted decision score (so ops can pick one). It reads against the schema from `mod-110-eval-data-platform-slice/exercise-01` and reuses the `v_verdict_full` view.

## 1. Solution overview

A multi-objective eval report is three artefacts stapled together, not one giant score:

1. **A Pareto frontier plot** across (quality, cost) and (quality, latency). Dominated candidates are visible at a glance; the frontier shows what is actually achievable.
2. **A weighted decision score** per candidate, with the weights published in the report (not hidden in a notebook). The weights encode the product team's willingness to trade — e.g. "we will pay 10% more per request to buy 5 points of quality".
3. **Slice tables** for cohorts that matter (tenant class, tool shape, locale). A candidate that wins in aggregate but loses on paid-tier traffic should not pass this report.

Five load-bearing decisions anchor the design:

1. **Axes are reported on their native units.** Quality in [0, 1] (or a judge score), cost in USD per 1k requests, latency in milliseconds p50/p95/p99. No normalisation inside the report — normalise only to compute the decision score, and show the normalisation step so a reader can argue with it.
2. **Decision score is *linear-with-caps*, not a soft product.** `score = w_q*norm_q - w_c*norm_c - w_l*norm_l`, with hard caps: if p95 latency > SLO, the candidate is marked INELIGIBLE regardless of score. Prevents a "cheap and fast but bad" candidate from winning.
3. **Confidence intervals on every axis.** Quality is a bootstrapped CI from the eval set. Cost and latency are percentile CIs from the live sample. Reporting `0.812` without `[0.791, 0.834]` is a lie by omission; two candidates whose CIs overlap are not meaningfully different.
4. **Paired comparison when the dataset allows it.** When all candidates ran on the *same* `dataset_version`, use a paired test (McNemar or paired bootstrap) instead of an unpaired difference. This typically shrinks the required sample size by 3–5× and is what makes the report decidable.
5. **Report is reproducible from `(dataset_version_id, candidate_ids)`.** The exact rows the plot was computed from are recoverable; a reviewer who doesn't trust the plot can run the same query and get the same numbers.

Candidates are rows in a small `candidate` table that points at the three artefacts that define a run under test: `app_version_id` (the deployable build), `prompt_version_id` (the prompt rev), and the implicit `model_pin` carried on the app version. Everything else — judge, dataset, tool shape — is held constant across candidates in a single report.

```
                 ┌────────────────────────────────────────────┐
                 │               multi-objective report       │
                 └────────────────────────────────────────────┘
                    ▲               ▲                 ▲
                    │               │                 │
   ┌────────────────┴────┐   ┌──────┴────────┐   ┌────┴────────────┐
   │  quality (CI)       │   │  cost (CI)    │   │ latency (CI)    │
   │  from eval_result   │   │  from         │   │ from trace      │
   │  .score, grouped    │   │  trace.cost   │   │ .duration_ns    │
   │  by app_version     │   │  + judge cost │   │                 │
   └────────────────────┘   └───────────────┘   └─────────────────┘
                    ▲               ▲                 ▲
                    └───────────┬───┴────────┬────────┘
                                │   same (dataset_version, judge_version)
                              candidate rows (app_version × prompt_version)
```

## 2. Worked implementation

### 2.1 `report/schemas.py` — report data types

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal

Axis = Literal["quality", "cost_per_1k_requests_usd", "latency_p95_ms"]


@dataclass(frozen=True)
class Candidate:
    candidate_id: str                 # human label, e.g. "gpt-4o-mini + prompt v7"
    app_version_id: str               # sha256:... from mod-110 ex-01
    prompt_version_id: str
    model_pin: str                    # for display only; truth is on app_version
    role: Literal["baseline", "challenger"]


@dataclass(frozen=True)
class AxisPoint:
    value: float
    ci_low: float
    ci_high: float
    n_samples: int
    unit: str                         # "score", "usd", "ms"


@dataclass(frozen=True)
class CandidateResult:
    candidate: Candidate
    quality: AxisPoint
    cost: AxisPoint
    latency_p50: AxisPoint
    latency_p95: AxisPoint
    latency_p99: AxisPoint
    eligibility: Literal["eligible", "latency_gated", "cost_gated", "error_gated"]
    eligibility_reason: str           # empty when eligible


@dataclass(frozen=True)
class DecisionWeights:
    w_quality: float                  # [0, 1]
    w_cost: float
    w_latency: float
    # Hard eligibility gates. A candidate failing a gate is NEVER selected.
    p95_latency_cap_ms: int
    cost_cap_usd_per_1k: float
    error_rate_cap: float

    def __post_init__(self) -> None:
        total = self.w_quality + self.w_cost + self.w_latency
        if abs(total - 1.0) > 1e-6:
            raise ValueError(f"weights must sum to 1.0, got {total}")


@dataclass(frozen=True)
class ReportInput:
    dataset_version_id: str           # exactly one; all candidates must have run on it
    judge_evaluator_version_id: str   # exactly one; keep the judge fixed
    candidates: list[Candidate]
    weights: DecisionWeights
    bootstrap_iters: int = 2000
    seed: int = 20241119
```

### 2.2 `report/sql/candidate_rows.sql` — the single source read

```sql
-- Returns one row per (candidate, trace) with the three raw axes attached.
-- Every candidate MUST have scored against the same dataset_version so the
-- comparison is paired. Candidates missing rows for a dataset row yield NULL
-- in `quality_score`; the report writer treats NULL quality as `status != ok`
-- and surfaces it as an eligibility problem, not a silent drop.
WITH candidate_ids AS (
    SELECT app_version_id, prompt_version_id, candidate_id
    FROM report_candidate
    WHERE report_id = $1
)
SELECT
    c.candidate_id,
    t.trace_id,
    t.duration_ns / 1e6                                             AS latency_ms,
    t.cost_usd                                                      AS app_cost_usd,
    er.cost_usd                                                     AS judge_cost_usd,
    er.score                                                        AS quality_score,
    er.status                                                       AS judge_status,
    t.status                                                        AS app_status,
    t.tenant_class, t.surface, t.tool_shape
FROM candidate_ids c
JOIN trace       t  ON t.app_version_id = c.app_version_id
                   AND t.prompt_version_id = c.prompt_version_id
LEFT JOIN eval_result er
                   ON er.trace_id = t.trace_id
                  AND er.evaluator_version_id = $2
                  AND er.dataset_version_id   = $3
WHERE t.ingested_ns > NOW_NS() - 7 * 24 * 3600 * 1_000_000_000;      -- bounded window
```

Two things worth noting:

- The LEFT JOIN keeps traces for which the judge failed or was skipped. Those rows are counted in the error-rate gate (§2.5) rather than silently dropped from the quality denominator.
- Judge cost and app cost are both carried through. The report presents app cost and total cost separately — a decision to add a judge in production should not hide behind "model cost went down".

### 2.3 `report/compute.py` — axis computation with bootstrapped CIs

```python
from __future__ import annotations
import math
import random
from statistics import quantiles
from typing import Iterable

from .schemas import AxisPoint, Candidate, CandidateResult, DecisionWeights, ReportInput


def _percentile(xs: list[float], p: float) -> float:
    """Simple percentile; p is in [0, 1]. Uses nearest-rank on sorted copy."""
    if not xs:
        return math.nan
    s = sorted(xs)
    k = max(0, min(len(s) - 1, int(math.ceil(p * len(s))) - 1))
    return s[k]


def _bootstrap_ci(
    xs: list[float],
    stat: callable,
    iters: int,
    seed: int,
    alpha: float = 0.05,
) -> tuple[float, float]:
    """Percentile bootstrap CI for an arbitrary statistic over xs."""
    rng = random.Random(seed)
    boots: list[float] = []
    n = len(xs)
    for _ in range(iters):
        sample = [xs[rng.randrange(n)] for _ in range(n)]
        boots.append(stat(sample))
    return _percentile(boots, alpha / 2), _percentile(boots, 1 - alpha / 2)


def axis_quality(scores: list[float], seed: int, iters: int) -> AxisPoint:
    clean = [s for s in scores if s is not None]
    if not clean:
        return AxisPoint(math.nan, math.nan, math.nan, 0, "score")
    mean = sum(clean) / len(clean)
    lo, hi = _bootstrap_ci(clean, lambda xs: sum(xs) / len(xs), iters, seed)
    return AxisPoint(mean, lo, hi, len(clean), "score")


def axis_cost_per_1k(app_cost: list[float], judge_cost: list[float], iters: int, seed: int) -> AxisPoint:
    """Sum app + judge cost, scale to per-1000-requests, bootstrap the mean."""
    total = [(a or 0.0) + (j or 0.0) for a, j in zip(app_cost, judge_cost)]
    if not total:
        return AxisPoint(math.nan, math.nan, math.nan, 0, "usd_per_1k")
    mean_per_req = sum(total) / len(total)
    lo, hi = _bootstrap_ci(total, lambda xs: sum(xs) / len(xs), iters, seed)
    return AxisPoint(mean_per_req * 1000, lo * 1000, hi * 1000, len(total), "usd_per_1k")


def axis_latency(latencies_ms: list[float], p: float, iters: int, seed: int) -> AxisPoint:
    if not latencies_ms:
        return AxisPoint(math.nan, math.nan, math.nan, 0, "ms")
    point = _percentile(latencies_ms, p)
    lo, hi = _bootstrap_ci(latencies_ms, lambda xs: _percentile(xs, p), iters, seed)
    return AxisPoint(point, lo, hi, len(latencies_ms), "ms")
```

### 2.4 `report/decision.py` — weights, gates, and the ranked table

```python
from __future__ import annotations
from .schemas import CandidateResult, DecisionWeights


def _normalise(value: float, lo: float, hi: float, invert: bool = False) -> float:
    """Min-max to [0, 1] within the candidate set. Invert for cost/latency."""
    if hi == lo:
        return 0.5
    x = (value - lo) / (hi - lo)
    return 1 - x if invert else x


def decide(results: list[CandidateResult], w: DecisionWeights) -> list[dict]:
    """
    Produce a ranked decision table. Each row contains:
      - raw axis values
      - normalised axis values (within the candidate set)
      - decision score (higher is better)
      - eligibility
    Caller renders this into the markdown report.
    """
    qs  = [r.quality.value          for r in results]
    cs  = [r.cost.value             for r in results]
    ls  = [r.latency_p95.value      for r in results]
    q_lo, q_hi = min(qs), max(qs)
    c_lo, c_hi = min(cs), max(cs)
    l_lo, l_hi = min(ls), max(ls)

    out = []
    for r in results:
        n_q = _normalise(r.quality.value,     q_lo, q_hi, invert=False)
        n_c = _normalise(r.cost.value,        c_lo, c_hi, invert=True)
        n_l = _normalise(r.latency_p95.value, l_lo, l_hi, invert=True)
        score = w.w_quality * n_q + w.w_cost * n_c + w.w_latency * n_l
        out.append({
            "candidate_id":   r.candidate.candidate_id,
            "quality":        r.quality.value,
            "quality_ci":     [r.quality.ci_low,  r.quality.ci_high],
            "cost_per_1k":    r.cost.value,
            "cost_ci":        [r.cost.ci_low,     r.cost.ci_high],
            "latency_p95_ms": r.latency_p95.value,
            "latency_p99_ms": r.latency_p99.value,
            "eligibility":    r.eligibility,
            "reason":         r.eligibility_reason,
            "score":          score,
        })
    # Rank: eligible first, then by score desc. Ties broken by quality.
    out.sort(key=lambda d: (
        0 if d["eligibility"] == "eligible" else 1,
        -d["score"],
        -d["quality"],
    ))
    for i, row in enumerate(out):
        row["rank"] = i + 1
    return out
```

### 2.5 `report/eligibility.py` — hard gates

```python
from __future__ import annotations
from .schemas import CandidateResult, DecisionWeights


def apply_gates(
    *,
    candidate,
    quality, cost, latency_p50, latency_p95, latency_p99,
    error_rate: float,
    w: DecisionWeights,
) -> CandidateResult:
    gated = "eligible"
    reason = ""
    # Hard caps are expressed as "if the CI *upper* bound is still better
    # than the cap, we accept; else, gated." This is the right side for
    # cost/latency; we want to be conservative.
    if latency_p95.ci_low > w.p95_latency_cap_ms:
        gated = "latency_gated"
        reason = f"p95 latency {latency_p95.value:.0f}ms (CI [{latency_p95.ci_low:.0f},{latency_p95.ci_high:.0f}]) exceeds SLO {w.p95_latency_cap_ms}ms"
    elif cost.ci_low > w.cost_cap_usd_per_1k:
        gated = "cost_gated"
        reason = f"cost ${cost.value:.3f}/1k (CI [{cost.ci_low:.3f},{cost.ci_high:.3f}]) exceeds budget ${w.cost_cap_usd_per_1k:.3f}/1k"
    elif error_rate > w.error_rate_cap:
        gated = "error_gated"
        reason = f"error rate {error_rate:.2%} exceeds cap {w.error_rate_cap:.2%}"
    return CandidateResult(
        candidate=candidate,
        quality=quality, cost=cost,
        latency_p50=latency_p50, latency_p95=latency_p95, latency_p99=latency_p99,
        eligibility=gated,
        eligibility_reason=reason,
    )
```

### 2.6 `report/render.py` — the markdown report

```python
from __future__ import annotations
from .schemas import ReportInput


MARKDOWN_TEMPLATE = """\
# Multi-objective eval report — {dataset_name}

- Report id: `{report_id}`
- Dataset version: `{dataset_version_id}`
- Judge version: `{judge_evaluator_version_id}`
- Window: `{window_start} → {window_end}`
- Weights: quality={w_q}, cost={w_c}, latency={w_l}
- Gates: p95 ≤ {cap_p95}ms, cost ≤ ${cap_cost}/1k, error ≤ {cap_err:.2%}

## Decision

**Recommended**: `{winner_id}` — score {winner_score:.3f}.

> Rationale: {rationale}

## Ranked candidates

| Rank | Candidate | Quality (95% CI) | Cost $/1k (CI) | p95 ms (CI) | p99 ms | Eligibility | Score |
|---|---|---|---|---|---|---|---|
{ranked_rows}

## Pareto frontier (quality vs cost)

{pareto_quality_cost_ascii}

## Pareto frontier (quality vs latency_p95)

{pareto_quality_latency_ascii}

## Slice tables

### By tenant_class

{slice_tenant_class}

### By tool_shape

{slice_tool_shape}

## Paired significance (vs baseline `{baseline_id}`)

| Candidate | Δquality | 95% CI on Δ | Paired bootstrap p | Decision |
|---|---|---|---|---|
{paired_rows}

## Methodology

- Quality statistic: bootstrapped mean (iters={iters}, seed={seed}) over judge scores.
- Cost: bootstrapped mean of per-request (app_cost + judge_cost) × 1000.
- Latency: bootstrapped p95/p99 over trace.duration_ns.
- Gates are evaluated against the CI, not the point estimate: a candidate is gated only when its CI bound is still over the cap.
- Candidates with judge_status != ok count toward the error-rate gate.
- Reproduce with:
  ```
  python -m report.main --report-id {report_id}
  ```

## Lineage

| Candidate | app_version_id | prompt_version_id | model_pin |
|---|---|---|---|
{lineage_rows}

## Links

- Platform SLOs: mod-110 ex-04 `v_slo_dashboard`.
- CI gate integration: mod-110 ex-05 (`gate_verdict`).
- Routing eval (next step): exercise-02 of this module.
"""
```

### 2.7 `report/main.py` — the CLI entry point

```python
from __future__ import annotations
import argparse
import json
import pathlib

from .compute   import axis_cost_per_1k, axis_latency, axis_quality
from .decision  import decide
from .eligibility import apply_gates
from .render    import MARKDOWN_TEMPLATE
from .schemas   import Candidate, CandidateResult, DecisionWeights, ReportInput
from .store     import load_candidate_rows, load_report_input


def _group(rows, key):
    out = {}
    for r in rows:
        out.setdefault(r[key], []).append(r)
    return out


def run(report_id: str) -> str:
    inp: ReportInput = load_report_input(report_id)
    rows = load_candidate_rows(report_id)
    grouped = _group(rows, "candidate_id")

    results: list[CandidateResult] = []
    for cand in inp.candidates:
        g = grouped.get(cand.candidate_id, [])
        q_scores   = [r["quality_score"] for r in g]
        a_costs    = [r["app_cost_usd"]  for r in g]
        j_costs    = [r["judge_cost_usd"] or 0.0 for r in g]
        latencies  = [r["latency_ms"]    for r in g]
        quality    = axis_quality(q_scores, inp.seed, inp.bootstrap_iters)
        cost       = axis_cost_per_1k(a_costs, j_costs, inp.bootstrap_iters, inp.seed)
        p50        = axis_latency(latencies, 0.50, inp.bootstrap_iters, inp.seed)
        p95        = axis_latency(latencies, 0.95, inp.bootstrap_iters, inp.seed)
        p99        = axis_latency(latencies, 0.99, inp.bootstrap_iters, inp.seed)
        err_rate   = sum(1 for r in g if r["judge_status"] != "ok" or r["app_status"] != "ok") / max(1, len(g))
        results.append(apply_gates(
            candidate=cand,
            quality=quality, cost=cost,
            latency_p50=p50, latency_p95=p95, latency_p99=p99,
            error_rate=err_rate, w=inp.weights,
        ))

    ranked = decide(results, inp.weights)
    winner = next((r for r in ranked if r["eligibility"] == "eligible"), None)
    md = MARKDOWN_TEMPLATE.format(
        report_id=report_id,
        # ... other template fields filled from inp/results ...
        winner_id=winner["candidate_id"] if winner else "NONE_ELIGIBLE",
        winner_score=winner["score"] if winner else 0.0,
        rationale=_rationale(winner, ranked, inp.weights) if winner else
                  "No candidate passed the hard gates — see Eligibility column.",
        ranked_rows=_render_ranked_rows(ranked),
        pareto_quality_cost_ascii=_ascii_pareto(ranked, "cost_per_1k", invert=True),
        pareto_quality_latency_ascii=_ascii_pareto(ranked, "latency_p95_ms", invert=True),
        # ...
    )
    return md


def _rationale(winner: dict, ranked: list[dict], w: DecisionWeights) -> str:
    runner_up = next((r for r in ranked if r["candidate_id"] != winner["candidate_id"]), None)
    if not runner_up:
        return "Only one eligible candidate."
    dq = winner["quality"]  - runner_up["quality"]
    dc = winner["cost_per_1k"] - runner_up["cost_per_1k"]
    dl = winner["latency_p95_ms"] - runner_up["latency_p95_ms"]
    return (
        f"Beats {runner_up['candidate_id']} by score {winner['score'] - runner_up['score']:+.3f} "
        f"(Δquality={dq:+.3f}, Δcost={dc:+.3f}$/1k, Δlatency_p95={dl:+.0f}ms). "
        f"Weights used: q={w.w_quality}, c={w.w_cost}, l={w.w_latency}."
    )


if __name__ == "__main__":
    parser = argparse.ArgumentParser()
    parser.add_argument("--report-id", required=True)
    parser.add_argument("--out", type=pathlib.Path, default=pathlib.Path("report.md"))
    args = parser.parse_args()
    args.out.write_text(run(args.report_id))
```

### 2.8 `report/paired.py` — paired bootstrap vs baseline

```python
from __future__ import annotations
import random


def paired_delta_ci(
    baseline_scores: list[tuple[str, float]],
    challenger_scores: list[tuple[str, float]],
    *,
    iters: int,
    seed: int,
    alpha: float = 0.05,
) -> dict:
    """Paired bootstrap on per-trace quality deltas.

    Inputs are lists of (trace_id, score). We inner-join on trace_id so only
    rows present in both candidates contribute. Returns Δ, CI, and a two-sided p.
    """
    b = dict(baseline_scores)
    c = dict(challenger_scores)
    pairs = [(b[t], c[t]) for t in b.keys() & c.keys()]
    if not pairs:
        return {"delta": 0.0, "ci": [0.0, 0.0], "p": 1.0, "n": 0}
    deltas = [cs - bs for bs, cs in pairs]
    mean = sum(deltas) / len(deltas)
    rng = random.Random(seed)
    boots = []
    for _ in range(iters):
        sample = [deltas[rng.randrange(len(deltas))] for _ in deltas]
        boots.append(sum(sample) / len(sample))
    boots.sort()
    lo = boots[int(len(boots) * alpha / 2)]
    hi = boots[int(len(boots) * (1 - alpha / 2))]
    # Two-sided p: fraction of boot means on the "wrong side" of 0.
    k = sum(1 for m in boots if (m <= 0.0) == (mean > 0))
    p = 2 * min(k, len(boots) - k) / len(boots)
    return {"delta": mean, "ci": [lo, hi], "p": p, "n": len(pairs)}
```

### 2.9 `tests/test_report.py` — smoke tests

```python
import math
import pytest

from report.compute     import axis_quality, axis_latency, axis_cost_per_1k
from report.decision    import decide
from report.eligibility import apply_gates
from report.paired      import paired_delta_ci
from report.schemas     import (
    Candidate, CandidateResult, DecisionWeights, AxisPoint,
)


def _mk_point(v, n=100): return AxisPoint(v, v - 0.01, v + 0.01, n, "u")


def test_quality_ci_contains_point():
    xs = [0.5 + 0.1 * (i % 5) for i in range(200)]
    p = axis_quality(xs, seed=1, iters=500)
    assert p.ci_low <= p.value <= p.ci_high
    assert p.n_samples == 200


def test_latency_bootstrap_returns_percentile_in_ci():
    xs = [float(i) for i in range(1, 1001)]
    p95 = axis_latency(xs, 0.95, iters=500, seed=2)
    assert 900 <= p95.value <= 1000
    assert p95.ci_low <= p95.value <= p95.ci_high


def test_weights_must_sum_to_one():
    with pytest.raises(ValueError):
        DecisionWeights(0.5, 0.3, 0.3, 500, 1.0, 0.01)


def test_eligibility_latency_gate_uses_ci_lower_bound():
    cand = Candidate("c", "sha256:a", "sha256:b", "gpt-4o-mini", "challenger")
    w = DecisionWeights(0.6, 0.2, 0.2, p95_latency_cap_ms=500,
                        cost_cap_usd_per_1k=5.0, error_rate_cap=0.01)
    hi_lat = AxisPoint(520.0, 510.0, 530.0, 300, "ms")    # whole CI over cap
    low_lat = AxisPoint(480.0, 470.0, 495.0, 300, "ms")   # whole CI under cap
    r1 = apply_gates(candidate=cand, quality=_mk_point(0.8), cost=_mk_point(1.0),
                     latency_p50=_mk_point(300), latency_p95=hi_lat,
                     latency_p99=_mk_point(700), error_rate=0.0, w=w)
    r2 = apply_gates(candidate=cand, quality=_mk_point(0.8), cost=_mk_point(1.0),
                     latency_p50=_mk_point(300), latency_p95=low_lat,
                     latency_p99=_mk_point(700), error_rate=0.0, w=w)
    assert r1.eligibility == "latency_gated"
    assert r2.eligibility == "eligible"


def test_paired_delta_only_uses_common_traces():
    baseline = [("t1", 0.5), ("t2", 0.6), ("t3", 0.7)]
    challenger = [("t2", 0.8), ("t3", 0.6)]
    out = paired_delta_ci(baseline, challenger, iters=500, seed=3)
    assert out["n"] == 2            # t1 excluded
    assert out["delta"] == pytest.approx((0.2 + -0.1) / 2)


def test_decide_breaks_ties_with_quality():
    cand_a = Candidate("A", "sha256:a1", "sha256:p1", "m", "challenger")
    cand_b = Candidate("B", "sha256:a2", "sha256:p1", "m", "challenger")
    w = DecisionWeights(0.6, 0.2, 0.2, 1000, 10, 0.5)
    results = [
        CandidateResult(cand_a, _mk_point(0.80), _mk_point(1.0),
                        _mk_point(100), _mk_point(300), _mk_point(500),
                        "eligible", ""),
        CandidateResult(cand_b, _mk_point(0.85), _mk_point(1.0),
                        _mk_point(100), _mk_point(300), _mk_point(500),
                        "eligible", ""),
    ]
    ranked = decide(results, w)
    assert ranked[0]["candidate_id"] == "B"


def test_gated_candidate_cannot_win_even_with_higher_score():
    # Score numerically bigger but latency gated.
    cand_gated   = Candidate("G", "sha256:g", "sha256:p", "m", "challenger")
    cand_ok      = Candidate("OK", "sha256:o", "sha256:p", "m", "challenger")
    w = DecisionWeights(0.6, 0.2, 0.2, p95_latency_cap_ms=500,
                        cost_cap_usd_per_1k=10.0, error_rate_cap=0.05)
    r_gated = apply_gates(
        candidate=cand_gated, quality=_mk_point(0.99), cost=_mk_point(0.1),
        latency_p50=_mk_point(300), latency_p95=AxisPoint(600, 590, 610, 100, "ms"),
        latency_p99=_mk_point(800), error_rate=0.0, w=w,
    )
    r_ok = apply_gates(
        candidate=cand_ok, quality=_mk_point(0.80), cost=_mk_point(1.0),
        latency_p50=_mk_point(300), latency_p95=_mk_point(400),
        latency_p99=_mk_point(600), error_rate=0.0, w=w,
    )
    ranked = decide([r_gated, r_ok], w)
    assert ranked[0]["candidate_id"] == "OK"
    assert ranked[1]["eligibility"] == "latency_gated"
```

### 2.10 Worked example report (abbreviated)

Given three candidates on dataset `d-2026-11-15` with weights `q=0.6, c=0.2, l=0.2`:

| Rank | Candidate | Quality (95% CI) | Cost $/1k (CI) | p95 ms (CI) | p99 ms | Eligibility | Score |
|---|---|---|---|---|---|---|---|
| 1 | gpt-4o-mini + prompt v7 | 0.821 [0.805, 0.836] | 1.42 [1.38, 1.47] | 540 [520, 561] | 790 | eligible | 0.743 |
| 2 | gpt-4o + prompt v7 | 0.869 [0.855, 0.882] | 7.80 [7.60, 8.02] | 820 [798, 844] | 1210 | **latency_gated** (SLO 700ms) | 0.681 |
| 3 | haiku-3.5 + prompt v7 | 0.772 [0.755, 0.789] | 0.52 [0.50, 0.55] | 340 [329, 352] | 480 | eligible | 0.701 |

> **Recommended**: `gpt-4o-mini + prompt v7`. Beats `haiku-3.5 + prompt v7` by score +0.042
> (Δquality=+0.049, Δcost=+0.90$/1k, Δlatency_p95=+200ms). `gpt-4o + prompt v7`
> has the best quality but is INELIGIBLE: its p95 latency CI is entirely over the 700ms SLO.

Paired vs baseline (`gpt-4o-mini + prompt v6`):

| Candidate | Δquality | 95% CI | p | Decision |
|---|---|---|---|---|
| gpt-4o-mini + prompt v7 | +0.019 | [+0.008, +0.030] | 0.004 | improvement |
| gpt-4o + prompt v7 | +0.067 | [+0.056, +0.079] | <0.001 | improvement (but gated) |
| haiku-3.5 + prompt v7 | -0.030 | [-0.041, -0.018] | <0.001 | regression |

## 3. Validation steps

1. **Run the test suite**: `pytest -q tests/test_report.py`. All seven tests pass.
2. **Run against a seeded fixture**: `python -m report.main --report-id fx-001 --out /tmp/r.md`. The output contains all seven template sections; the "Recommended" block names a candidate or explicitly states no eligible candidate.
3. **Dataset sanity**: every candidate in the report input has `COUNT(trace) ≥ 100` on the declared `dataset_version_id`; the report refuses to run with `n < 30` per candidate.
4. **CI coverage**: re-run with `bootstrap_iters=4000` and `seed=20241120`; the point estimate moves < 0.5% and the CI bounds move < 1%. The report is seed-stable.
5. **Gate-trip test**: set `p95_latency_cap_ms` below all candidates' p95 lower bounds; every candidate becomes `latency_gated`; the recommendation reads `NONE_ELIGIBLE`.
6. **Paired determinism**: run the paired bootstrap twice with the same seed; the Δ, CI, and p are identical to the last decimal.
7. **Lineage**: pick any candidate in the report; verify the `(app_version_id, prompt_version_id)` pair exists in `app_version` and `prompt_version` tables (mod-110 ex-01).
8. **Reproducibility hash**: store a SHA256 of `(report_id, sorted candidate_ids, dataset_version_id, judge_version_id, weights, seed)` on the rendered report; two runs with the same inputs produce the same hash.
9. **Slice coverage**: the "by tenant_class" and "by tool_shape" tables list every value present in the trace set, including values with `n < 30` (annotated `(low n)` instead of being hidden).
10. **Human read-through**: a product manager who did not write the report should be able to point at the recommendation and say which weight knob to turn to flip it. If the report doesn't support that question, the weights section is doing too little work.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|---|---|---|
| R1 | Three axes reported on native units with confidence intervals. | Quality, cost, p95/p99 latency each have point + CI. | Only point estimates; or axes collapsed to a single compound score. |
| R2 | Hard gates (latency, cost, error rate) are first-class and precede the ranked table. | `eligibility` column present; a gated candidate cannot be `Recommended`. | Score is used without gating; "cheap+fast+bad" candidate wins. |
| R3 | Decision weights are declared in the report body, not a notebook. | Weights appear in header; `sum == 1`. | Weights hidden in code; reader cannot audit. |
| R4 | Paired comparison used when all candidates share a `dataset_version`. | Paired bootstrap on per-trace deltas. | Independent-sample difference with inflated variance. |
| R5 | Slice tables cover tenant_class, tool_shape, locale (≥ 2 of 3). | Slice section present with per-slice winner column. | Only aggregate numbers; a cohort regression is invisible. |
| R6 | Cost separates app cost from judge cost. | Two columns or at least an "incl. judge" note. | Judge cost silently folded into app cost. |
| R7 | Report is reproducible from a `report_id` + git sha. | `main.py --report-id` produces an identical byte-for-byte report. | Numbers depend on ambient time or current table state. |
| R8 | Pareto frontier shown for at least (quality, cost) and (quality, latency). | Two frontier plots or ASCII equivalents. | No visualisation of trade-offs. |
| R9 | Lineage table ties every candidate to `app_version_id` and `prompt_version_id`. | Lineage section present. | Candidate IDs are free-text only. |
| R10 | Statistical significance reported alongside differences. | Paired Δ, CI, p (or equivalent) per challenger. | Deltas reported without CI or test. |

A passing submission is 8+ of 10 with R1, R2, R3, R4, R7 all hard-required.

## 5. Common mistakes

- **Collapsing to a single score too early.** The headline score is useful; replacing the three raw axes with it is not. Readers need to see the trade-off, not just the result of the trade-off.
- **No confidence intervals.** Two candidates whose CIs overlap substantially are not meaningfully different, no matter how far apart their point estimates. A report without CIs will convince a reader to ship a coin flip.
- **Soft gates for hard constraints.** Latency SLO and error-rate cap are not weights. If p95 > SLO, the candidate is ineligible — fold that into the weighted score and you will eventually ship something that violates its own SLO.
- **Averaging p95s.** You cannot average percentiles across groups and get a global p95. Compute the global percentile from the raw sample, or compute per-cohort percentiles and report them separately.
- **Unpaired tests on paired data.** When all candidates ran the same dataset, use per-trace paired differences — the variance is typically 3–5× smaller and the test needs 3–5× fewer rows to detect the same effect.
- **Judge cost hidden in app cost.** A candidate that "wins on cost" may have won because the new judge is cheaper. Keep judge and app costs separate; one of them is a product decision, the other is an eval-plane decision.
- **Dataset drift inside the report window.** If the report pulls `last 7 days` of traces across all candidates but candidate A only shipped on day 6, the comparison is apples-to-oranges. Enforce a shared time window or run all candidates offline on the same `dataset_version`.
- **No tenant-class slice.** Aggregate metrics can look fine while paid-tier traffic regresses. A report that doesn't slice by tenant class will miss revenue-affecting regressions.
- **Treating latency as one number.** p50 moves for user-perceived warmth; p95 and p99 move for SLO burn. Reporting only mean latency hides the tails that pager calls come from.
- **Over-smoothing with too many bootstrap iters on too few rows.** 10,000 iters on 20 rows produces confident-looking CIs that are still noise. Report `n_samples` and refuse to publish below a documented floor.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 2.3` (metric coverage), `MEASURE 2.7` (measurement validity), `MANAGE 1.3` (trade-off documentation).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML04` (membership inference) motivates the per-tenant slice; a model that memorises one cohort shows up as an outlier here.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0043` (craft adversarial input) frames why robustness slices are reported alongside aggregate quality.
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/>, for the `gen_ai.usage.*`, `gen_ai.response.model`, and duration attributes consumed by this report.
- Local exercise reader: `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-01-multi-objective-eval-report.md`.
- Related solutions: `mod-110-eval-data-platform-slice/exercise-01` (schema), `mod-110/exercise-04` (SLOs the latency cap reads from), `mod-104-llm-as-judge-in-product` (judge contract), and this module's exercise-02 (routing eval builds on this report's cohort design).
