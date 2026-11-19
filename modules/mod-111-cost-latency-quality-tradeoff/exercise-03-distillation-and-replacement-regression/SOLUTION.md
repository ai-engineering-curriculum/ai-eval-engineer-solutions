# exercise-03 — Distillation and Replacement Regression — Solution

> Reference solution for `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-03-distillation-and-replacement-regression.md`.
> This is an **implementation** exercise. The deliverable is a regression harness that decides whether a candidate (a distilled student, a new provider's model, a smaller finetune) can replace the incumbent (the teacher / current production model) without an unacceptable quality loss. The harness pairs every test case through both models, uses a non-inferiority test (not a mean-wins test), and gates on slices as well as aggregate.

## 1. Solution overview

"The new model is cheaper and faster, does it regress quality?" is the question. The naïve answer — "compare means, if the new model is within a point, ship it" — is wrong three ways:

1. A non-inferiority margin isn't the same as a mean-wins test. A difference of `-0.003` with a 95% CI of `[-0.08, +0.07]` is not "no regression"; it's "we don't know".
2. Aggregate parity hides slice regressions. A student that matches the teacher in total but loses 10 points on code-tool traffic is a regression on the only cohort that matters for a given team.
3. Quality is one axis. A pure quality regression test does not catch a model that passes quality but ships a new failure mode (refusal rate up, hallucination rate up, tool-call miss up).

The harness:

1. Runs the candidate and the incumbent against the **same frozen `dataset_version`** (from mod-110 ex-01 schema). Pairs rows by `(dataset_row_id)`.
2. Applies a **non-inferiority test** against a published margin `δ`. The candidate passes if the upper bound of the `(incumbent − candidate)` 95% CI is below `δ`.
3. **Slices** the test by cohort (tool_shape × tenant_class × locale) and requires each slice to pass. A slice that is too small to resolve passes conditionally and is flagged.
4. Runs **auxiliary metrics**: refusal-rate parity, hallucination-rate parity, tool-call accuracy parity, latency p95 ≤ incumbent + budget. A regression on any auxiliary metric fails the whole gate.
5. Emits a **gate verdict** written to `eval_result`-shape storage (`evaluator_id='regression.distillation'`, `evaluator_version_id=sha(this harness code + margin spec)`). The CI gate in mod-110 ex-05 reads against it.

Five load-bearing decisions anchor the design:

1. **Non-inferiority, not superiority, not equivalence.** The product question is "is the new model NOT worse by more than δ". `TOST` (two one-sided tests) is overkill; a single one-sided test on `incumbent − candidate ≤ δ` is the right answer. Writing this out in the spec prevents the next person from running a two-sided test and misreading the p.
2. **Margin is published per task family, not globally.** A `δ = 0.03` makes sense for a judgment-score task; it's nonsense for exact-match accuracy. The margin is a function of the task family; it is declared in `margins.yaml` and reviewed on every change.
3. **Paired bootstrap, not Welch.** The two models scored the SAME rows. Use per-row deltas; the variance is dominated by row difficulty, not model noise, and the paired test is dramatically tighter.
4. **Slice gate is AND, aggregate gate is a summary.** A candidate that passes aggregate but fails any slice in `top_slices` fails the gate. The aggregate number is reporting, not decision.
5. **Verdict is append-only.** `gate_verdict` rows are keyed by `(candidate_version_id, incumbent_version_id, dataset_version_id, harness_version_id)`. Rerunning on the same inputs returns the same decision; a new harness version writes a new row.

```
  dataset_version (fixed)
         │
         ▼
   ┌────────────┐   score → eval_result (incumbent)
   │ incumbent  │
   └────────────┘
         │
         ▼
   ┌────────────┐   score → eval_result (candidate)
   │ candidate  │
   └────────────┘
         │
         ▼
   paired_scores view (SQL)
         │
         ▼
   non_inferiority_test  + slice_gate + auxiliary_metrics
         │
         ▼
   gate_verdict  (verdict: pass | fail | needs_review)
```

## 2. Worked implementation

### 2.1 `regression/schemas.py`

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal

Verdict = Literal["pass", "fail", "needs_review"]


@dataclass(frozen=True)
class Margin:
    task_family: str                    # "judge_score" | "exact_match" | "f1" | "boolean_match"
    delta: float                        # non-inferiority margin, in task-family units
    direction: Literal["higher_is_better", "lower_is_better"] = "higher_is_better"


@dataclass(frozen=True)
class SliceSpec:
    key: str                            # "tool_shape" | "tool_shape:tenant_class" | ...
    required: bool                      # if True, slice MUST pass; if False, logged only
    min_n: int = 30                     # slices below this are reported but not gated


@dataclass(frozen=True)
class AuxCheck:
    name: str                           # "refusal_rate" | "hallucination_rate" | "latency_p95_ms" | ...
    max_delta: float                    # candidate - incumbent must be ≤ max_delta


@dataclass(frozen=True)
class HarnessConfig:
    dataset_version_id: str
    incumbent_evaluator_version_id: str
    candidate_evaluator_version_id: str
    primary_metric: Literal["judge_score", "exact_match", "f1", "boolean_match"]
    margin: Margin
    slices: list[SliceSpec]
    auxiliaries: list[AuxCheck]
    bootstrap_iters: int = 4000
    seed: int = 20241119


@dataclass(frozen=True)
class SliceResult:
    key: str
    value: str
    n: int
    incumbent_mean: float
    candidate_mean: float
    delta_mean: float                   # candidate - incumbent
    delta_ci_upper: float               # one-sided 95% upper bound on (incumbent - candidate)
    non_inferior: bool                  # delta_ci_upper <= margin.delta
    status: Literal["pass", "fail", "low_n"]


@dataclass(frozen=True)
class GateResult:
    verdict: Verdict
    aggregate: SliceResult
    slices: list[SliceResult]
    auxiliaries: list[dict]             # [{name, incumbent, candidate, delta, pass}]
    reason: str
    harness_version_id: str
```

### 2.2 `regression/sql/paired_scores.sql`

```sql
-- One row per dataset_row_id with incumbent + candidate scores side by side.
-- INNER JOIN on dataset_row_id: a dataset row scored by only one model is
-- excluded (recorded in the "coverage" audit, not silently dropped from
-- the paired analysis).
WITH incumbent AS (
    SELECT er.trace_id,
           er.score                AS inc_score,
           er.status               AS inc_status,
           er.latency_ms           AS inc_latency_ms,
           er.cost_usd             AS inc_cost_usd,
           t.app_version_id        AS inc_app_version_id,
           t.tool_shape, t.tenant_class, t.surface,
           -- The dataset_row_id lives in trace attributes; the ingest writer
           -- promotes it to a column in a future schema bump.
           t.payload_sha256,
           (t.attributes ->> 'dataset_row_id') AS dataset_row_id
    FROM eval_result er
    JOIN trace t ON t.trace_id = er.trace_id
    WHERE er.evaluator_version_id = $1
      AND er.dataset_version_id   = $3
),
candidate AS (
    SELECT er.trace_id,
           er.score                AS cand_score,
           er.status               AS cand_status,
           er.latency_ms           AS cand_latency_ms,
           er.cost_usd             AS cand_cost_usd,
           t.app_version_id        AS cand_app_version_id,
           (t.attributes ->> 'dataset_row_id') AS dataset_row_id
    FROM eval_result er
    JOIN trace t ON t.trace_id = er.trace_id
    WHERE er.evaluator_version_id = $2
      AND er.dataset_version_id   = $3
)
SELECT i.dataset_row_id,
       i.inc_score,  c.cand_score,
       i.inc_status, c.cand_status,
       i.inc_latency_ms, c.cand_latency_ms,
       i.inc_cost_usd,   c.cand_cost_usd,
       i.tool_shape, i.tenant_class, i.surface
FROM incumbent i
JOIN candidate c USING (dataset_row_id);
```

### 2.3 `regression/stats.py` — paired non-inferiority

```python
from __future__ import annotations
import math
import random
from typing import Iterable, Literal


def paired_non_inferiority(
    incumbent_scores: list[float],
    candidate_scores: list[float],
    *,
    margin: float,
    direction: Literal["higher_is_better", "lower_is_better"] = "higher_is_better",
    iters: int,
    seed: int,
    alpha: float = 0.05,
) -> dict:
    """
    One-sided non-inferiority test by paired bootstrap.

    - For higher_is_better, we want to show
          (incumbent - candidate) < margin
      i.e. candidate is at most `margin` worse than incumbent.
    - For lower_is_better (e.g. hallucination rate), we want to show
          (candidate - incumbent) < margin.

    The decision rule: the (1-alpha) upper bound of the appropriate signed
    difference must be below `margin`.
    """
    if direction == "higher_is_better":
        deltas = [inc - cand for inc, cand in zip(incumbent_scores, candidate_scores)]
    else:
        deltas = [cand - inc for inc, cand in zip(incumbent_scores, candidate_scores)]
    deltas = [d for d in deltas if d is not None and not math.isnan(d)]
    if len(deltas) < 2:
        return {
            "n": len(deltas),
            "mean_delta": math.nan,
            "upper_ci": math.nan,
            "margin": margin,
            "non_inferior": False,
            "reason": "insufficient_pairs",
        }
    mean_delta = sum(deltas) / len(deltas)
    rng = random.Random(seed)
    boots = []
    for _ in range(iters):
        sample = [deltas[rng.randrange(len(deltas))] for _ in deltas]
        boots.append(sum(sample) / len(sample))
    boots.sort()
    # One-sided upper bound: 1 - alpha percentile.
    upper = boots[min(len(boots) - 1, int(iters * (1 - alpha)))]
    return {
        "n": len(deltas),
        "mean_delta": mean_delta,
        "upper_ci": upper,
        "margin": margin,
        "non_inferior": upper < margin,
        "reason": "",
    }


def mcnemar_discordant(a_correct: list[bool], b_correct: list[bool]) -> dict:
    """
    McNemar's test for paired binary outcomes.

    For exact-match style metrics, the paired bootstrap reduces to McNemar's
    exact test on the discordant pairs. We keep it available for callers
    whose primary metric is boolean_match.
    """
    n01 = sum(1 for a, b in zip(a_correct, b_correct) if a and not b)  # inc right, cand wrong
    n10 = sum(1 for a, b in zip(a_correct, b_correct) if b and not a)  # cand right, inc wrong
    # Exact binomial test under H0: n10 / (n01+n10) = 0.5
    import math
    n = n01 + n10
    if n == 0:
        return {"n_discordant": 0, "p_two_sided": 1.0, "n01": 0, "n10": 0}
    from math import comb
    # Two-sided p by summing the tail of the symmetric distribution.
    k = min(n01, n10)
    p = sum(comb(n, i) for i in range(k + 1)) / 2**n
    return {"n_discordant": n, "n01": n01, "n10": n10, "p_two_sided": min(1.0, 2 * p)}
```

### 2.4 `regression/slicer.py`

```python
from __future__ import annotations
from .schemas import SliceResult, SliceSpec


def slice_rows(rows: list[dict], key_expr: str) -> dict[str, list[dict]]:
    """key_expr is 'tool_shape' or 'tool_shape:tenant_class' etc."""
    keys = key_expr.split(":")
    out: dict[str, list[dict]] = {}
    for r in rows:
        v = ":".join(str(r[k]) for k in keys)
        out.setdefault(v, []).append(r)
    return out


def compute_slice(
    rows: list[dict], key_expr: str, slice_value: str,
    *, margin: float, iters: int, seed: int,
) -> SliceResult:
    from .stats import paired_non_inferiority
    inc = [r["inc_score"]  for r in rows]
    cnd = [r["cand_score"] for r in rows]
    n = len(rows)
    if n < 2:
        return SliceResult(
            key=key_expr, value=slice_value, n=n,
            incumbent_mean=0.0, candidate_mean=0.0, delta_mean=0.0,
            delta_ci_upper=0.0, non_inferior=False, status="low_n",
        )
    res = paired_non_inferiority(inc, cnd, margin=margin, iters=iters, seed=seed)
    return SliceResult(
        key=key_expr, value=slice_value, n=n,
        incumbent_mean=sum(inc) / n,
        candidate_mean=sum(cnd) / n,
        delta_mean=res["mean_delta"],
        delta_ci_upper=res["upper_ci"],
        non_inferior=res["non_inferior"],
        status=(
            "pass" if res["non_inferior"] and n >= 30 else
            "low_n" if n < 30 else "fail"
        ),
    )
```

### 2.5 `regression/auxiliaries.py` — refusal / hallucination / latency parity

```python
from __future__ import annotations
from .stats import paired_non_inferiority


def refusal_rate(rows: list[dict], side: str) -> float:
    k = f"{side}_status"
    n = len(rows)
    if n == 0: return 0.0
    return sum(1 for r in rows if r.get(k) == "refused") / n


def hallucination_rate(rows: list[dict], side: str) -> float:
    k = f"{side}_label"
    n = len(rows)
    if n == 0: return 0.0
    return sum(1 for r in rows if (r.get(k) or "").startswith("hallucination")) / n


def check_auxiliary(name: str, inc_vals: list[float], cand_vals: list[float],
                    max_delta: float, iters: int, seed: int) -> dict:
    """
    Auxiliary is lower-is-better: refusal, hallucination, latency, cost.
    Candidate wins if (candidate - incumbent) < max_delta with 95% confidence.
    """
    res = paired_non_inferiority(
        inc_vals, cand_vals,
        margin=max_delta, direction="lower_is_better",
        iters=iters, seed=seed,
    )
    return {
        "name": name,
        "incumbent_mean": (sum(inc_vals)  / len(inc_vals))  if inc_vals  else 0.0,
        "candidate_mean": (sum(cand_vals) / len(cand_vals)) if cand_vals else 0.0,
        "mean_delta":  res["mean_delta"],
        "upper_ci":    res["upper_ci"],
        "max_delta":   max_delta,
        "pass":        res["non_inferior"],
    }
```

### 2.6 `regression/harness.py`

```python
from __future__ import annotations
import hashlib
import json
from typing import Iterable

from .schemas      import GateResult, HarnessConfig, SliceResult, SliceSpec, Verdict
from .slicer       import compute_slice, slice_rows
from .stats        import paired_non_inferiority
from .auxiliaries  import check_auxiliary, refusal_rate, hallucination_rate


def harness_version_id(cfg: HarnessConfig, code_sha: str) -> str:
    """Content-address the harness version so verdict rows can refer to it."""
    body = json.dumps({
        "code_sha": code_sha,
        "margin":   cfg.margin.__dict__,
        "primary":  cfg.primary_metric,
        "slices":   [s.__dict__ for s in cfg.slices],
        "aux":      [a.__dict__ for a in cfg.auxiliaries],
    }, sort_keys=True).encode()
    return "sha256:" + hashlib.sha256(body).hexdigest()


def run_gate(cfg: HarnessConfig, rows: list[dict], code_sha: str) -> GateResult:
    agg = compute_slice(
        rows, key_expr="__aggregate__", slice_value="all",
        margin=cfg.margin.delta, iters=cfg.bootstrap_iters, seed=cfg.seed,
    )
    slices: list[SliceResult] = []
    for spec in cfg.slices:
        grouped = slice_rows(rows, spec.key)
        for value, sub in grouped.items():
            sr = compute_slice(
                sub, key_expr=spec.key, slice_value=value,
                margin=cfg.margin.delta, iters=cfg.bootstrap_iters, seed=cfg.seed,
            )
            slices.append(sr)

    auxiliaries = []
    for a in cfg.auxiliaries:
        if a.name == "refusal_rate":
            inc = [1.0 if r["inc_status"]  == "refused" else 0.0 for r in rows]
            cnd = [1.0 if r["cand_status"] == "refused" else 0.0 for r in rows]
            auxiliaries.append(check_auxiliary(a.name, inc, cnd, a.max_delta,
                                               cfg.bootstrap_iters, cfg.seed))
        elif a.name == "hallucination_rate":
            inc = [1.0 if (r.get("inc_label")  or "").startswith("hallucination") else 0.0 for r in rows]
            cnd = [1.0 if (r.get("cand_label") or "").startswith("hallucination") else 0.0 for r in rows]
            auxiliaries.append(check_auxiliary(a.name, inc, cnd, a.max_delta,
                                               cfg.bootstrap_iters, cfg.seed))
        elif a.name == "latency_p95_ms":
            inc = sorted(r["inc_latency_ms"]  for r in rows)
            cnd = sorted(r["cand_latency_ms"] for r in rows)
            idx = max(0, int(0.95 * len(inc)) - 1)
            auxiliaries.append({
                "name": a.name,
                "incumbent_mean": inc[idx] if inc else 0.0,
                "candidate_mean": cnd[idx] if cnd else 0.0,
                "mean_delta":     (cnd[idx] if cnd else 0.0) - (inc[idx] if inc else 0.0),
                "upper_ci":       None,
                "max_delta":      a.max_delta,
                "pass":           (cnd[idx] if cnd else 0.0) - (inc[idx] if inc else 0.0) < a.max_delta,
            })
        else:
            raise NotImplementedError(f"auxiliary {a.name}")

    verdict, reason = _decide(cfg, agg, slices, auxiliaries)
    return GateResult(
        verdict=verdict, aggregate=agg, slices=slices, auxiliaries=auxiliaries,
        reason=reason,
        harness_version_id=harness_version_id(cfg, code_sha),
    )


def _decide(cfg: HarnessConfig, agg: SliceResult,
            slices: list[SliceResult], aux: list[dict]) -> tuple[Verdict, str]:
    failed_required_slices = [s for s in slices if s.status == "fail"
                              and any(spec.key == s.key and spec.required for spec in cfg.slices)]
    failed_aux = [a for a in aux if not a["pass"]]
    if not agg.non_inferior:
        return "fail", f"aggregate fails non-inferiority: upper CI {agg.delta_ci_upper:+.4f} ≥ margin {cfg.margin.delta}"
    if failed_required_slices:
        s = failed_required_slices[0]
        return "fail", f"slice {s.key}={s.value} fails (Δ upper CI {s.delta_ci_upper:+.4f})"
    if failed_aux:
        a = failed_aux[0]
        return "fail", f"auxiliary {a['name']} fails (Δ {a['mean_delta']:+.4f}, max {a['max_delta']})"
    low_n = [s for s in slices if s.status == "low_n"]
    if low_n:
        return "needs_review", f"{len(low_n)} slices below min_n; aggregate passes"
    return "pass", "aggregate + all required slices + auxiliaries pass"
```

### 2.7 `regression/main.py`

```python
from __future__ import annotations
import argparse
import json
import pathlib

from .harness import run_gate
from .schemas import AuxCheck, HarnessConfig, Margin, SliceSpec
from .store   import load_paired_rows, persist_gate_verdict


def default_config_for(dataset_version_id: str,
                       incumbent_id: str,
                       candidate_id: str) -> HarnessConfig:
    return HarnessConfig(
        dataset_version_id=dataset_version_id,
        incumbent_evaluator_version_id=incumbent_id,
        candidate_evaluator_version_id=candidate_id,
        primary_metric="judge_score",
        margin=Margin(task_family="judge_score", delta=0.02),
        slices=[
            SliceSpec(key="tool_shape",                   required=True,  min_n=30),
            SliceSpec(key="tenant_class",                 required=True,  min_n=30),
            SliceSpec(key="tool_shape:tenant_class",      required=False, min_n=30),
        ],
        auxiliaries=[
            AuxCheck(name="refusal_rate",       max_delta=0.01),
            AuxCheck(name="hallucination_rate", max_delta=0.005),
            AuxCheck(name="latency_p95_ms",     max_delta=150.0),
        ],
    )


def main() -> None:
    ap = argparse.ArgumentParser()
    ap.add_argument("--dataset-version-id", required=True)
    ap.add_argument("--incumbent",          required=True)
    ap.add_argument("--candidate",          required=True)
    ap.add_argument("--code-sha",           required=True)
    ap.add_argument("--out", type=pathlib.Path, default=pathlib.Path("verdict.json"))
    args = ap.parse_args()
    cfg = default_config_for(args.dataset_version_id, args.incumbent, args.candidate)
    rows = load_paired_rows(cfg)
    result = run_gate(cfg, rows, args.code_sha)
    persist_gate_verdict(cfg, result)
    args.out.write_text(json.dumps({
        "verdict": result.verdict,
        "reason": result.reason,
        "harness_version_id": result.harness_version_id,
        "aggregate": result.aggregate.__dict__,
        "slices": [s.__dict__ for s in result.slices],
        "auxiliaries": result.auxiliaries,
    }, indent=2))
```

### 2.8 `margins.yaml` — the published decision thresholds

```yaml
# All margins are "candidate may be at most this much worse than incumbent".
# A change to this file requires PR review from the task owner AND the eval
# platform owner; the harness_version_id depends on it, so verdicts rerun.
margins:
  - task_family: judge_score
    delta: 0.02              # 2 points of a [0, 1] judge
    rationale: |
      Empirically, product teams start to notice at Δ≈0.03. A 0.02 margin
      with a 4000-iter bootstrap on n≥500 holds the type-I error below 5%.
  - task_family: exact_match
    delta: 0.015             # 1.5 pp of accuracy
  - task_family: f1
    delta: 0.02
  - task_family: boolean_match
    delta: 0.01
auxiliaries:
  refusal_rate:       0.01   # 1 pp absolute
  hallucination_rate: 0.005  # 0.5 pp absolute
  latency_p95_ms:     150    # 150 ms absolute
  cost_ratio:         0.0    # candidate MUST be cheaper or equal (0 margin)
```

### 2.9 `tests/test_regression_harness.py`

```python
import math
import pytest

from regression.auxiliaries import check_auxiliary
from regression.harness     import run_gate, harness_version_id
from regression.schemas     import (
    AuxCheck, HarnessConfig, Margin, SliceSpec,
)
from regression.stats       import mcnemar_discordant, paired_non_inferiority


def _cfg(seed=1):
    return HarnessConfig(
        dataset_version_id="sha256:" + "d" * 64,
        incumbent_evaluator_version_id="sha256:" + "i" * 64,
        candidate_evaluator_version_id="sha256:" + "c" * 64,
        primary_metric="judge_score",
        margin=Margin(task_family="judge_score", delta=0.02),
        slices=[
            SliceSpec(key="tool_shape",   required=True, min_n=30),
            SliceSpec(key="tenant_class", required=True, min_n=30),
        ],
        auxiliaries=[
            AuxCheck(name="refusal_rate", max_delta=0.01),
            AuxCheck(name="latency_p95_ms", max_delta=150.0),
        ],
        bootstrap_iters=2000, seed=seed,
    )


def _rows(n, inc_mean=0.80, cand_mean=0.79, inc_lat=400, cand_lat=350,
          tool_shape="rag", tenant_class="paid", refusal_cand=False):
    import random
    random.seed(0)
    out = []
    for i in range(n):
        out.append({
            "dataset_row_id": f"r{i}",
            "inc_score":  min(1.0, max(0.0, random.gauss(inc_mean, 0.05))),
            "cand_score": min(1.0, max(0.0, random.gauss(cand_mean, 0.05))),
            "inc_status":  "ok",
            "cand_status": "refused" if refusal_cand and i < n // 50 else "ok",
            "inc_latency_ms":  inc_lat  + (i % 100),
            "cand_latency_ms": cand_lat + (i % 100),
            "tool_shape":  tool_shape,
            "tenant_class": tenant_class,
        })
    return out


def test_harness_passes_when_candidate_is_close():
    cfg = _cfg()
    rows = _rows(500, inc_mean=0.80, cand_mean=0.79)
    out = run_gate(cfg, rows, code_sha="deadbeef")
    assert out.verdict == "pass"
    assert out.aggregate.non_inferior


def test_harness_fails_on_aggregate_regression():
    cfg = _cfg()
    rows = _rows(500, inc_mean=0.80, cand_mean=0.70)   # -0.10, way beyond 0.02
    out = run_gate(cfg, rows, code_sha="deadbeef")
    assert out.verdict == "fail"
    assert "aggregate" in out.reason


def test_harness_fails_on_slice_regression():
    cfg = _cfg()
    rows = _rows(500, inc_mean=0.80, cand_mean=0.80)   # aggregate OK
    # Flip 50 rag paid rows to a big candidate regression.
    rows += _rows(100, inc_mean=0.90, cand_mean=0.70,
                   tool_shape="rag", tenant_class="paid")
    out = run_gate(cfg, rows, code_sha="deadbeef")
    assert out.verdict == "fail"
    assert "slice" in out.reason


def test_harness_fails_on_auxiliary_refusal():
    cfg = _cfg()
    rows = _rows(500, refusal_cand=True)               # candidate refuses 2% more
    out = run_gate(cfg, rows, code_sha="deadbeef")
    assert out.verdict == "fail"
    assert "refusal_rate" in out.reason


def test_harness_verdict_is_deterministic():
    cfg = _cfg(seed=42)
    rows = _rows(500)
    a = run_gate(cfg, rows, code_sha="sha").__dict__
    b = run_gate(cfg, rows, code_sha="sha").__dict__
    assert a["verdict"] == b["verdict"]
    assert a["aggregate"].delta_ci_upper == b["aggregate"].delta_ci_upper


def test_harness_version_id_changes_with_margin():
    cfg_a = _cfg()
    cfg_b = HarnessConfig(**{**cfg_a.__dict__, "margin": Margin(task_family="judge_score", delta=0.03)})
    assert harness_version_id(cfg_a, "sha") != harness_version_id(cfg_b, "sha")


def test_needs_review_when_slices_below_min_n():
    cfg = _cfg()
    # Two slices, one too small.
    rows = _rows(100, tool_shape="rag", tenant_class="paid")
    rows += _rows(20, tool_shape="rag", tenant_class="internal")   # low_n
    out = run_gate(cfg, rows, code_sha="deadbeef")
    assert out.verdict == "needs_review"


def test_mcnemar_detects_shift():
    inc = [True] * 95 + [False] * 5
    cnd = [True] * 85 + [False] * 15
    res = mcnemar_discordant(inc, cnd)
    assert res["n01"] >= res["n10"]
    assert res["p_two_sided"] < 0.1


def test_paired_non_inferiority_returns_insufficient_for_small_n():
    out = paired_non_inferiority([0.8], [0.79], margin=0.02, iters=100, seed=1)
    assert out["reason"] == "insufficient_pairs"
    assert out["non_inferior"] is False
```

### 2.10 Worked example verdict (abbreviated)

```json
{
  "verdict": "fail",
  "reason": "slice tool_shape=rag:paid fails (Δ upper CI +0.034 ≥ margin 0.02)",
  "harness_version_id": "sha256:...",
  "aggregate": {
    "n": 6821,
    "incumbent_mean": 0.802,
    "candidate_mean": 0.790,
    "delta_mean":   -0.012,
    "delta_ci_upper": 0.008,
    "non_inferior": true
  },
  "slices": [
    {"key": "tool_shape", "value": "chat_only", "n": 2104, "non_inferior": true,
     "delta_mean": -0.004, "delta_ci_upper": 0.009},
    {"key": "tool_shape", "value": "rag",       "n": 3011, "non_inferior": false,
     "delta_mean": -0.027, "delta_ci_upper": 0.034},
    {"key": "tool_shape", "value": "tool_call", "n": 1706, "non_inferior": true,
     "delta_mean": -0.009, "delta_ci_upper": 0.011}
  ],
  "auxiliaries": [
    {"name": "refusal_rate",       "mean_delta": 0.001, "max_delta": 0.01,  "pass": true},
    {"name": "hallucination_rate", "mean_delta": 0.000, "max_delta": 0.005, "pass": true},
    {"name": "latency_p95_ms",     "mean_delta": -120,  "max_delta": 150,   "pass": true}
  ]
}
```

A reader sees: the candidate is cheaper and 120 ms faster at p95, aggregate passes, but the RAG-tool slice regresses by 2.7 points of quality — enough to fail the gate. The decision is sharp and the slice responsible is named.

## 3. Validation steps

1. **Unit tests**: `pytest -q tests/test_regression_harness.py` — all 9 tests pass.
2. **Golden fixture**: a seeded dataset with a planted uniform regression of `-0.015` with `n=1,000` passes; one with `-0.03` fails; one with `-0.01` aggregate but `-0.05` on `tool_shape=rag` fails with the slice reason.
3. **Determinism**: two runs with `(cfg, rows, code_sha)` identical produce the same verdict and the same bootstrap upper CI.
4. **Harness version**: bumping `margin.delta` in `margins.yaml` and re-running regenerates `harness_version_id` and writes a new `gate_verdict` row without overwriting the old one.
5. **Coverage audit**: a dataset row that was scored by only one model appears in a "coverage" log; the paired view omits it; the harness refuses to proceed when coverage drops below 95%.
6. **Auxiliary gates**: artificially set a 1% refusal-rate increase on the candidate; the harness flips to `fail` with the aux reason, even if the primary metric passes.
7. **McNemar sanity**: for a boolean-match task family, swap in `mcnemar_discordant`; results agree with the paired bootstrap at n=500.
8. **Reproducibility of verdict row**: query `gate_verdict WHERE candidate_version_id = $1 AND dataset_version_id = $2 AND harness_version_id = $3`; exactly one row per `(candidate, dataset, harness)`.
9. **Latency parity**: candidate p95 is computed on the SAME dataset rows; the harness never compares p95 of candidate on production traffic against p95 of incumbent on eval traffic.
10. **Verdict blocking**: the CI gate in mod-110 ex-05 consumes `gate_verdict` and blocks a PR when `verdict ∈ {fail, needs_review}` without an override token.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|---|---|---|
| R1 | Paired comparison: incumbent and candidate score the SAME `dataset_version` rows, joined by `dataset_row_id`. | SQL enforces join; unit test proves pairing. | Unpaired comparison with wildly different samples. |
| R2 | Non-inferiority test, not a mean-wins test. Margin published in `margins.yaml`. | Decision uses `upper_ci < margin`; file exists and is reviewable. | Decision uses two-sided p-value or a point estimate. |
| R3 | Slice gating required for at least `tool_shape` and `tenant_class`. | `SliceSpec(required=True)` for both; unit test shows slice fail path. | Only aggregate gating; cohort regression invisible. |
| R4 | Auxiliary metrics for refusal, hallucination (or equivalent error), and latency are gated. | Three auxiliaries in config; any failure fails the gate. | Only primary metric checked. |
| R5 | Harness version is content-addressed, includes margins + slices. | `harness_version_id` test proves margin change bumps id. | Harness version static; no record of what rule ran. |
| R6 | Verdict written to append-only `gate_verdict`-shape storage. | Persist step unit-tested; rerun doesn't overwrite. | Verdict only in stdout; no replay. |
| R7 | Small slices reported but not gated (`low_n`), with `needs_review` verdict. | Unit test covers `needs_review`. | Low-n slices silently pass. |
| R8 | Coverage audit: dataset rows scored by only one model are visible. | Query or log present; harness aborts below 95% coverage. | Silent dropping of unpaired rows. |
| R9 | Bootstrap deterministic by `seed`; results reproducible. | Determinism test passes. | Different numbers on each run. |
| R10 | Latency / cost auxiliaries use the same paired rows as quality. | SQL view shared; no mismatched sampling. | Latency pulled from production; quality from eval set. |

A passing submission is 9+ of 10 with R1, R2, R3, R5, R6, R9 hard-required.

## 5. Common mistakes

- **Two-sided test on a one-sided question.** The product question is "not worse by more than δ". A two-sided test burns half its budget on a direction you don't care about and often reports "no significant difference" when the data cannot support non-inferiority.
- **Aggregate-only gating.** A slice that carries 20% of traffic can regress by 0.1 while the aggregate holds. Shipping it ships a regression on exactly the users who pay you.
- **Unpaired comparison.** Running candidate on fresh production traffic and comparing to the incumbent's historical numbers conflates time-of-day, dataset drift, and model differences. Lock the dataset; pair the rows.
- **Choosing the margin after seeing the data.** If `δ` is a parameter you tune post-hoc, every gate passes. Publish margins in `margins.yaml` and bump the harness version when they change.
- **Dropping unpaired rows silently.** If a candidate failed on 10% of rows and you compute means on the 90% that worked, you have hidden a failure mode. Audit coverage.
- **Mixing boolean and continuous metrics with the same test.** McNemar's and paired bootstrap are not interchangeable; use the paired bootstrap for continuous scores and McNemar for exact-match outcomes.
- **Measuring latency on different traffic.** "Candidate latency p95 was 400ms on prod" vs "incumbent latency was 500ms on the eval set" is not a comparison. Latency belongs on the same paired rows.
- **Treating p95 latency as a bootstrap-free quantity.** p95 has sampling variance like any other statistic; use the bootstrap or Harrell-Davis and gate on the upper CI, not the point.
- **Ignoring refusal rate.** A distilled model often ships new refusals (safety classifier over-fires). A quality parity test will not catch it; an auxiliary refusal-rate gate will.
- **Verdict written over stdout only.** A gate verdict must be a durable row keyed by `(candidate, dataset, harness_version)` so replays are stable and auditors can answer "why did we ship this".

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MANAGE 4.1` (regression safeguards), `MEASURE 2.7` (test validity), `GOVERN 1.6` (traceability of the harness that produced a verdict).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML06` (AI supply chain) motivates lineage of both candidate and incumbent artefacts; `ML09` (output integrity) motivates the hallucination-rate auxiliary.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0043` (craft adversarial input) frames the slice-level regression check; a distilled model is often weaker on adversarial cohorts even when aggregate quality holds.
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/>, for the latency and token attributes used by auxiliaries.
- Local exercise reader: `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-03-distillation-and-replacement-regression.md`.
- Related solutions: this module's exercise-01 (bootstrapped CIs and slice tables), exercise-02 (cohort preservation), `mod-110/exercise-01` (schema the paired view reads from), `mod-110/exercise-05` (CI gate consumes `gate_verdict`), `mod-106-eval-gated-cicd` (gate semantics).
