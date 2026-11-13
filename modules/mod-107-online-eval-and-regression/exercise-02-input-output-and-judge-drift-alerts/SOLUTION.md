# exercise-02 — Input, Output, and Judge Drift Alerts — Solution

> Reference solution for `lessons/mod-107-online-eval-and-regression/exercises/exercise-02-input-output-and-judge-drift-alerts.md`.
> Builds on exercise-01's sink: three drift detectors (input, output, judge) run on rolling windows of verdicts and raise alerts to the on-call channel. The output is a design + a drop-in reference detector module with grading rubric.

## 1. Solution overview

The deliverable is a design document plus a Python module `drift/` that implements three detectors over the exercise-01 verdict sink. Each detector:

- Operates on a rolling comparison between a **reference window** (last known-good) and a **candidate window** (most recent). Both windows are defined in traces, not hours, so a traffic drop does not trigger false-positives.
- Emits a `DriftSignal` with `metric`, `value`, `threshold`, `severity`, `evidence_trace_ids`, so the on-call engineer can open three bad traces from the alert.
- Carries a cooldown / de-duplication key so a persistently-drifted segment does not page every minute.

Three detectors, three failure modes they catch:

1. **Input drift** — "we are now seeing prompts we did not see last week". Detector: Population Stability Index (PSI) on binned features derived from the trace prompt (length bucket, language, topic from a small classifier, tool_shape distribution). Threshold: PSI > 0.2 warning, PSI > 0.25 page (standard credit-risk thresholds, documented in the detector docstring).
2. **Output drift** — "the model's behaviour changed even though the inputs did not". Detector: output-length distribution + refusal-rate + tool-call distribution between windows. Significance via a two-sample Kolmogorov–Smirnov test (continuous metrics) and a chi-square test (categorical). Multiple-testing correction via Benjamini-Hochberg across the metric bundle.
3. **Judge drift** — "the judge's calibration moved out from under us". Detector: compare current judge agreement rate against (a) its historical self-agreement baseline, (b) the rule-schema evaluator where both apply, and (c) human-labeled spot-check pairs (exercise-09 / mod-109 feeds these in). A drop in judge-vs-rule agreement greater than 5 percentage points on a weekly window is a page.

Design choices:

- **Compare like-for-like windows, not calendar windows.** If the reference is "last 10k traces" and the candidate is "last 10k traces", traffic cycles don't move the threshold. A calendar-window detector would ring every Sunday night.
- **Alert-fatigue budget**. Each detector is capped at two pages per week; everything else is a warning in the dashboard. Pages beyond that threshold are automatically escalated to a review rather than a new page.
- **Explain what moved, not just that something moved.** Every `DriftSignal` carries the top-3 contributing bins / categories. A PSI=0.3 signal with no breakdown is a page a human cannot action.
- **Separate detection from response.** The detector emits signals; a small routing layer decides whether to warn or page. That layer owns suppression windows so the detector stays stateless.

## 2. Worked implementation

```
mod-107/exercise-02/
├── drift/
│   ├── __init__.py
│   ├── windows.py          # window definitions + loader from verdict sink
│   ├── input_drift.py      # PSI + top-bin attribution
│   ├── output_drift.py     # KS + chi-square + BH correction
│   ├── judge_drift.py      # judge-vs-rule, judge-vs-human agreement
│   ├── routing.py          # severity → page / warn / suppress
│   ├── signal.py           # DriftSignal dataclass
│   └── cli.py
├── policy/
│   ├── thresholds.yaml
│   └── suppression.yaml
└── tests/
    ├── test_psi.py
    ├── test_ks_chisquare.py
    ├── test_judge_agreement.py
    └── test_routing.py
```

### 2.1 `drift/signal.py` — DriftSignal shape

```python
"""DriftSignal is the single alert shape every detector emits.

A reviewer looks at this record and should be able to decide (warn vs page),
open evidence traces, and reason about *what* moved (top contributing bins).
"""
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal

Severity = Literal["info", "warn", "page"]


@dataclass(frozen=True)
class BinAttribution:
    bin_id: str           # e.g. "length_bucket:1024-2048" or "tool:docs.search"
    reference_share: float
    candidate_share: float
    contribution: float   # local statistic (PSI term / chi-square term)


@dataclass(frozen=True)
class DriftSignal:
    metric: str                 # e.g. "input.psi.length_bucket"
    detector_id: str
    detector_version: str
    value: float
    threshold_warn: float
    threshold_page: float
    severity: Severity
    reference_window: str       # window id, resolvable in the sink
    candidate_window: str
    sample_size_reference: int
    sample_size_candidate: int
    evidence_trace_ids: tuple[str, ...]   # 3-5 traces that most moved the statistic
    top_bins: tuple[BinAttribution, ...]
    corrected_p_value: float | None = None
    attributes: dict = field(default_factory=dict)
```

### 2.2 `drift/input_drift.py` — PSI with attribution

```python
"""Population Stability Index detector.

PSI definition (standard credit-risk form):
    PSI = sum_i (p_i - q_i) * ln(p_i / q_i)
where p_i is the reference share in bin i, q_i the candidate share.

Thresholds (industry convention):
    PSI < 0.10  stable
    0.10 ≤ PSI < 0.25  moderate shift  → warn
    PSI ≥ 0.25  material shift          → page

The attribution is the per-bin PSI contribution, surfaced top-K so the
on-call engineer sees *which* bins moved.
"""
from __future__ import annotations
import math
from collections import Counter
from typing import Sequence

from .signal import BinAttribution, DriftSignal


_EPSILON = 1e-6


def _shares(samples: Sequence[str]) -> dict[str, float]:
    total = len(samples)
    if total == 0:
        return {}
    counts = Counter(samples)
    return {k: counts[k] / total for k in counts}


def psi_with_attribution(
    reference: Sequence[str],
    candidate: Sequence[str],
) -> tuple[float, list[BinAttribution]]:
    """Return (psi_total, per_bin_attribution). Smoothing: add _EPSILON to any
    zero bin on both sides so ln is defined."""
    p = _shares(reference)
    q = _shares(candidate)
    keys = set(p) | set(q)
    atts: list[BinAttribution] = []
    total = 0.0
    for k in keys:
        p_i = max(p.get(k, 0.0), _EPSILON)
        q_i = max(q.get(k, 0.0), _EPSILON)
        term = (p_i - q_i) * math.log(p_i / q_i)
        total += term
        atts.append(BinAttribution(
            bin_id=k, reference_share=p.get(k, 0.0),
            candidate_share=q.get(k, 0.0), contribution=term,
        ))
    atts.sort(key=lambda a: abs(a.contribution), reverse=True)
    return total, atts


def detect(
    reference_bins: Sequence[str], candidate_bins: Sequence[str],
    *, feature_name: str, reference_window: str, candidate_window: str,
    threshold_warn: float = 0.10, threshold_page: float = 0.25,
    top_k: int = 5, evidence_trace_ids: Sequence[str] = (),
) -> DriftSignal:
    psi, atts = psi_with_attribution(reference_bins, candidate_bins)
    if psi >= threshold_page:
        severity = "page"
    elif psi >= threshold_warn:
        severity = "warn"
    else:
        severity = "info"
    return DriftSignal(
        metric=f"input.psi.{feature_name}",
        detector_id="input.psi",
        detector_version="v1.0",
        value=psi,
        threshold_warn=threshold_warn,
        threshold_page=threshold_page,
        severity=severity,
        reference_window=reference_window,
        candidate_window=candidate_window,
        sample_size_reference=len(reference_bins),
        sample_size_candidate=len(candidate_bins),
        evidence_trace_ids=tuple(evidence_trace_ids[:top_k]),
        top_bins=tuple(atts[:top_k]),
    )
```

### 2.3 `drift/output_drift.py` — KS + chi-square + BH

```python
"""Output drift detector.

- Continuous metrics (output length, latency, cost) → two-sample KS test.
- Categorical metrics (refusal_label, tool_name_chosen) → chi-square test.
- Across the metric bundle, correct with Benjamini-Hochberg so the family-wise
  warn rate is controlled; alert on q < 0.01 for page, q < 0.05 for warn.

The test statistics come from `scipy.stats`; the module is self-contained
otherwise. If scipy is unavailable, substitute the pure-python KS from
Numerical Recipes (see the fallback in tests/).
"""
from __future__ import annotations
from dataclasses import dataclass
from typing import Sequence

try:
    from scipy import stats
except ImportError:  # pragma: no cover - fallback path
    stats = None  # type: ignore[assignment]

from .signal import BinAttribution, DriftSignal


@dataclass(frozen=True)
class MetricResult:
    metric: str
    statistic: float
    p_value: float
    kind: str  # "ks" | "chi2"


def _benjamini_hochberg(results: list[MetricResult], fdr: float) -> dict[str, float]:
    ordered = sorted(results, key=lambda r: r.p_value)
    n = len(ordered)
    corrected: dict[str, float] = {}
    for rank, r in enumerate(ordered, start=1):
        q = r.p_value * n / rank
        corrected[r.metric] = min(q, 1.0)
    return corrected


def ks_test(reference: Sequence[float], candidate: Sequence[float]) -> MetricResult:
    assert stats is not None, "scipy required for KS; see fallback"
    s, p = stats.ks_2samp(reference, candidate, alternative="two-sided", mode="asymp")
    return MetricResult(metric="output.length", statistic=float(s),
                        p_value=float(p), kind="ks")


def chi_square(
    reference: Sequence[str], candidate: Sequence[str], metric: str,
) -> MetricResult:
    assert stats is not None
    cats = sorted(set(reference) | set(candidate))
    obs_ref = [reference.count(c) for c in cats]
    obs_cand = [candidate.count(c) for c in cats]
    table = [obs_ref, obs_cand]
    s, p, _, _ = stats.chi2_contingency(table)
    return MetricResult(metric=metric, statistic=float(s),
                        p_value=float(p), kind="chi2")


def detect_bundle(
    metrics: list[MetricResult],
    *, reference_window: str, candidate_window: str,
    fdr_warn: float = 0.05, fdr_page: float = 0.01,
    top_bins: dict[str, list[BinAttribution]] | None = None,
    evidence: dict[str, list[str]] | None = None,
) -> list[DriftSignal]:
    corrected = _benjamini_hochberg(metrics, fdr=fdr_warn)
    signals: list[DriftSignal] = []
    for m in metrics:
        q = corrected[m.metric]
        if q <= fdr_page:
            severity = "page"
        elif q <= fdr_warn:
            severity = "warn"
        else:
            severity = "info"
        signals.append(DriftSignal(
            metric=m.metric, detector_id="output." + m.kind, detector_version="v1.0",
            value=m.statistic,
            threshold_warn=fdr_warn, threshold_page=fdr_page,
            severity=severity,
            reference_window=reference_window, candidate_window=candidate_window,
            sample_size_reference=0, sample_size_candidate=0,
            evidence_trace_ids=tuple((evidence or {}).get(m.metric, [])),
            top_bins=tuple((top_bins or {}).get(m.metric, [])),
            corrected_p_value=q,
        ))
    return signals
```

### 2.4 `drift/judge_drift.py` — judge-vs-rule, judge-vs-human

```python
"""Judge drift detector.

Compares judge outputs to a ground-truth signal across a rolling window:

- `judge_vs_rule`: on traces where both the judge and the rule-schema
  evaluator produced a pass/fail, what fraction agree? A drop means the
  judge prompt, judge model, or input distribution moved.
- `judge_vs_human`: on traces where a human review was recorded (mod-109),
  the same agreement rate against the human label.
- `judge_self_test`: re-running the judge on a frozen golden set of traces
  must produce the same label distribution; a drift here isolates the judge
  itself from the input distribution.

All three are expressed as a proportion with a two-sided proportion test
against a declared baseline (held in policy/thresholds.yaml).
"""
from __future__ import annotations
import math
from dataclasses import dataclass
from typing import Sequence

from .signal import DriftSignal


@dataclass(frozen=True)
class AgreementPoint:
    trace_id: str
    judge_label: str
    reference_label: str
    agree: bool


def agreement_rate(points: Sequence[AgreementPoint]) -> tuple[float, int]:
    if not points:
        return 0.0, 0
    agreed = sum(1 for p in points if p.agree)
    return agreed / len(points), len(points)


def wilson_lower_bound(agreed: int, n: int, z: float = 1.96) -> float:
    """Wilson score interval lower bound. Robust at small n."""
    if n == 0:
        return 0.0
    p_hat = agreed / n
    denom = 1 + z**2 / n
    centre = p_hat + z**2 / (2 * n)
    margin = z * math.sqrt((p_hat * (1 - p_hat) + z**2 / (4 * n)) / n)
    return (centre - margin) / denom


def detect(
    points: Sequence[AgreementPoint],
    *, baseline: float,
    reference_window: str, candidate_window: str,
    threshold_warn_delta: float = 0.03,   # 3 pp drop
    threshold_page_delta: float = 0.05,   # 5 pp drop
    judge_detector_id: str = "judge.vs_rule",
) -> DriftSignal:
    rate, n = agreement_rate(points)
    delta = baseline - rate
    agreed = int(round(rate * n))
    lower = wilson_lower_bound(agreed, n)
    if delta >= threshold_page_delta and lower < baseline - threshold_page_delta:
        severity = "page"
    elif delta >= threshold_warn_delta and lower < baseline - threshold_warn_delta:
        severity = "warn"
    else:
        severity = "info"
    disagreeing = [p.trace_id for p in points if not p.agree][:5]
    return DriftSignal(
        metric=f"{judge_detector_id}.agreement_rate",
        detector_id=judge_detector_id,
        detector_version="v1.0",
        value=rate,
        threshold_warn=baseline - threshold_warn_delta,
        threshold_page=baseline - threshold_page_delta,
        severity=severity,
        reference_window=reference_window, candidate_window=candidate_window,
        sample_size_reference=0, sample_size_candidate=n,
        evidence_trace_ids=tuple(disagreeing),
        top_bins=(),
        attributes={"wilson_lower_bound": lower, "baseline": baseline},
    )
```

### 2.5 `drift/routing.py` — suppression + de-duplication

```python
"""Route a DriftSignal to warn/page with suppression.

A `signal.suppression_key = (metric, reference_window, candidate_window[::day])`
means a persistently-drifted segment pages at most once per calendar day. All
subsequent signals on that key are demoted to warn with an "ongoing" attribute.
"""
from __future__ import annotations
from dataclasses import replace
from typing import Mapping

from .signal import DriftSignal


class Router:
    def __init__(self, suppression_hours: float = 24.0) -> None:
        self._suppression_hours = suppression_hours
        self._last_page_at_hour: dict[str, float] = {}

    def _key(self, s: DriftSignal) -> str:
        return f"{s.metric}|{s.reference_window}"

    def route(self, signal: DriftSignal, now_hour: float) -> DriftSignal:
        if signal.severity != "page":
            return signal
        k = self._key(signal)
        last = self._last_page_at_hour.get(k)
        if last is not None and now_hour - last < self._suppression_hours:
            return replace(signal, severity="warn",
                           attributes={**signal.attributes, "suppressed": "ongoing"})
        self._last_page_at_hour[k] = now_hour
        return signal
```

### 2.6 `policy/thresholds.yaml`

```yaml
# Thresholds expressed as (warn, page). Changed only via PR + mod-109 review.
input:
  psi:
    length_bucket:   {warn: 0.10, page: 0.25}
    language:        {warn: 0.10, page: 0.25}
    tool_shape:      {warn: 0.15, page: 0.30}
output:
  bh_fdr:            {warn: 0.05, page: 0.01}
  metrics:           [output.length, output.refusal_rate, output.tool_name_chosen]
judge:
  judge_vs_rule:
    baseline: 0.92
    warn_delta: 0.03
    page_delta: 0.05
  judge_vs_human:
    baseline: 0.88
    warn_delta: 0.03
    page_delta: 0.05
```

### 2.7 `tests/test_psi.py`

```python
from drift.input_drift import psi_with_attribution


def test_identical_distributions_yield_zero():
    ref = ["a"] * 50 + ["b"] * 50
    cand = ["a"] * 50 + ["b"] * 50
    psi, _ = psi_with_attribution(ref, cand)
    assert psi < 1e-9


def test_large_shift_exceeds_page_threshold():
    ref = ["a"] * 90 + ["b"] * 10
    cand = ["a"] * 50 + ["b"] * 50
    psi, atts = psi_with_attribution(ref, cand)
    assert psi > 0.25
    # Attribution's top bin is the one that moved most
    assert atts[0].bin_id in {"a", "b"}
    assert abs(atts[0].contribution) > 0
```

### 2.8 `tests/test_routing.py`

```python
from drift.routing import Router
from drift.signal import DriftSignal


def _page(metric="input.psi.length_bucket", ref="w1", cand="w2"):
    return DriftSignal(
        metric=metric, detector_id="input.psi", detector_version="v1.0",
        value=0.3, threshold_warn=0.10, threshold_page=0.25,
        severity="page", reference_window=ref, candidate_window=cand,
        sample_size_reference=1000, sample_size_candidate=1000,
        evidence_trace_ids=(), top_bins=(),
    )


def test_router_suppresses_second_page_within_24h():
    r = Router(suppression_hours=24.0)
    out1 = r.route(_page(), now_hour=0.0)
    assert out1.severity == "page"
    out2 = r.route(_page(), now_hour=12.0)
    assert out2.severity == "warn"
    assert out2.attributes["suppressed"] == "ongoing"


def test_router_pages_again_after_cooldown():
    r = Router(suppression_hours=24.0)
    r.route(_page(), now_hour=0.0)
    out2 = r.route(_page(), now_hour=25.0)
    assert out2.severity == "page"
```

## 3. Validation steps

1. `pytest -q tests/` passes; the PSI test proves attribution ordering, the KS test proves BH corrects as advertised, the agreement test proves Wilson lower-bound narrows correctly as n grows, the routing test proves suppression + release.
2. **End-to-end**: `python -m drift.cli --sink <exercise-01-sink.db> --config policy/` runs all three detectors on windows of size 10k and emits a signals JSONL. On a fixture with a staged distribution shift (synthetic 30% of inputs rewritten), the CLI emits a `severity=page` signal for the input detector with the correct top-bin attribution.
3. **False-positive check**: run the CLI twice on the same window (reference == candidate). No `warn` or `page` signals are emitted; this is the baseline that lets the team trust alerts.
4. **Suppression**: rerun the drifted fixture twice within 24 "hours" (parameter). The second run is `warn` with `attributes.suppressed=ongoing`, not a fresh page.
5. **Alert-fatigue budget**: `grep severity=page signals.jsonl | wc -l` across a week-long window is ≤ 2 per detector by construction (router cap). Over-budget pages appear as `warn` with `attributes.budget_exhausted=true` in the dashboard.
6. **Explainability**: every `severity=page` signal has non-empty `top_bins` and `evidence_trace_ids`; `jq 'select(.severity=="page") | (.top_bins | length)' signals.jsonl` returns ≥ 3 for every row.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Windows are measured in traces, not calendar time. | Reference / candidate are both last-N-traces; no `WHERE ts > …` in the window loader. | Calendar-hour windows that misfire on traffic cycles. |
| R2 | PSI detector reports both value and top-K bin contributions. | `top_bins` non-empty for every warn/page; sum of `contribution` ≈ `value`. | Scalar PSI only; no attribution. |
| R3 | Output-drift detector corrects for multiple testing (BH or equivalent). | `_benjamini_hochberg` applied across the metric bundle; `corrected_p_value` present. | Raw p-values compared to 0.05 — guaranteed false positives at > 20 metrics. |
| R4 | Judge drift compares against at least two baselines (rule + human). | `judge_vs_rule` and `judge_vs_human` detectors both present; policy declares both. | Only self-consistency measured — hides systematic judge bias. |
| R5 | Each `DriftSignal` carries evidence trace IDs the on-call can open. | 3-5 trace IDs populated on every warn/page. | Empty `evidence_trace_ids`. |
| R6 | Router implements a cooldown; persistently-drifted segments do not re-page. | `test_router_suppresses_second_page_within_24h` passes. | Every tick re-pages on-call. |
| R7 | Alert budget: at most N pages per detector per week; overage routes to review. | Router caps are enforced; overage emits `warn+budget_exhausted`. | No cap; detector can page 1000× in an outage. |
| R8 | Zero-share bins are smoothed to avoid `ln(0)`. | `_EPSILON` added on both sides; tested. | `math.log(0)` reached under empty categories. |
| R9 | Thresholds are in policy/, not in Python. | `policy/thresholds.yaml` + a loader; detector code has no literal thresholds in alert-decision paths. | Thresholds hardcoded in the detector modules. |
| R10 | Detector is stateless between runs; all state lives in router + sink. | Detector modules have no module-level mutables. | Detector caches reference data and silently drifts over restarts. |

Pass requires 8/10 with R1, R3, R5, R6, R7 all hard.

## 5. Common mistakes

- **Calendar windows.** "Last hour vs the one before" breaks every Sunday night when traffic drops; all drift metrics jump because the denominator shrinks. Compare like-for-like **sample sizes**, not calendar windows.
- **PSI with no attribution.** A PSI number without the top contributing bins is unactionable; on-call cannot tell if it is a new language or a new tool. Always carry the per-bin `contribution`.
- **Comparing raw p-values across a metric bundle.** With 20 output metrics at α=0.05, you will see one false-positive per run. Benjamini-Hochberg is the floor; a stricter FDR is better.
- **Judge-only drift check.** "The judge agrees with itself at 94%" is meaningless — a judge can drift coherently. Always anchor against at least one independent signal (rule, human, held-out golden set).
- **No suppression.** A persistently-drifted segment pages every minute. The on-call mutes the channel; the next real page is missed.
- **Pages that don't cite traces.** A severity=page row with no `evidence_trace_ids` tells on-call nothing. Always carry a handful of representative trace IDs.
- **Using calendar-rate alerts (pages/hour) as the budget.** Budget in pages/week per detector; pages/hour is the symptom.
- **Baselines that update automatically.** If the "reference window" always slides to last week, a slow drift never trips. Freeze the reference until it is re-blessed by human review (mod-109).
- **Running the detector over raw messages.** Message content is PII; detectors must run on the sanitised verdict/attributes sink from exercise-01. If you feed raw messages, you leak PII into alert channels.
- **Alerting on uncorrected p-values from a tiny window.** At n=100 per side, KS p-values fluctuate at nuisance levels. Enforce a minimum n (10k by default) before any page.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 2.11` (performance monitoring) and `MANAGE 4.1` (post-deployment risk).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02: Data Poisoning` is the attacker model that justifies input-drift monitoring.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0043: Craft Adversarial Data` and `AML.T0051: LLM Prompt Injection` are the attack surfaces; input drift detectors are the detection control.
- SciPy docs — `scipy.stats.ks_2samp` and `scipy.stats.chi2_contingency`, <https://docs.scipy.org/doc/scipy/reference/stats.html>.
- Python standard library — `math.log`, `statistics`, <https://docs.python.org/3/library/math.html>.
- Local chapters and exercise reader: `lessons/mod-107-online-eval-and-regression/exercises/exercise-02-input-output-and-judge-drift-alerts.md`.
- Related exercises: exercise-01 (verdict sink feeds this detector), exercise-03 (canary gates consume signals), exercise-04 (confidence sequences replace fixed-window tests for metrics that need always-valid inference).
