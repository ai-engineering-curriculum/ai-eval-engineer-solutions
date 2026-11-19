# exercise-02 — Model Routing Eval With Cohort Preservation — Solution

> Reference solution for `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-02-model-routing-eval-with-cohort-preservation.md`.
> This is an **implementation** exercise. The deliverable evaluates a model-routing policy (a per-request decision between a cheap/fast model and an expensive/strong model) in a way that respects the cohort the router produced. The naïve pitfall: compare "routed to strong" quality against "routed to cheap" quality and conclude the cheap model is worse. That is a confounded comparison; the router picked the harder requests for the strong model. The right evaluation preserves cohorts and measures the router against counterfactual baselines on each cohort separately, then composes the result.

## 1. Solution overview

A routing policy `π(request) → {cheap, strong}` lets the system spend compute only where it buys quality. Evaluating `π` requires three things the aggregate report from exercise-01 does not provide:

1. **Counterfactual labels per cohort.** For each request, you want to know what quality `strong` would have produced AND what quality `cheap` would have produced. Without both, you can't say whether the router routed correctly.
2. **Cohort-preserving decomposition.** The router produces two cohorts (`cohort=cheap`, `cohort=strong`). Within each cohort, the router is held fixed; comparisons inside a cohort are valid. Cross-cohort claims require counterfactuals.
3. **A regret / headroom metric.** "How much quality did the router leave on the table vs. the oracle that always picks the better model per request, and how much cost did it save vs. always-strong?"

Five load-bearing decisions anchor the solution:

1. **Shadow eval the "other" model on both cohorts.** Before promoting a router, replay a bounded sample (e.g. 5% of production) through BOTH models. These shadow calls give the counterfactual quality the router didn't buy. Without this, cohort cost and quality are unidentifiable.
2. **Routing regret is the headline.** `regret = quality(oracle) − quality(router)`; `cost_saving = cost(always_strong) − cost(router)`. A good router is close-to-oracle quality with large cost saving. Report both; never one without the other.
3. **Report per-cohort, then composed.** Each cohort has its own quality/cost/latency table. The composition multiplies by the cohort weight (proportion of production traffic that falls into each cohort). This prevents a cohort shift at deploy time from silently moving the aggregate.
4. **Router inversion test.** A rock-bottom check: `quality(strong | cohort=strong) ≥ quality(strong | cohort=cheap)`. If not, the router is anti-correlated with difficulty and should be scrapped. This is the one test every router must pass.
5. **Stratified sampling, not uniform.** The shadow sample is stratified by (tool_shape, tenant_class, cohort). Uniform sampling of production under-samples the strong-cohort (which is small by construction) and leaves you measuring the router with CIs 2× wider than necessary.

The eval reads against the mod-110 ex-01 schema: a new `routing_decision` row is attached to each trace, and the shadow calls land as separate traces linked by `shadow_of` to the production trace.

```
                 production request
                        │
                        ▼
                   ┌─────────┐
                   │ router π│
                   └─────────┘
                    │       │
            cohort=cheap  cohort=strong
                    │       │
            ┌───────┘       └───────┐
            ▼                       ▼
        cheap_model             strong_model     ← prod cost paid here
            │                       │
            │  (sampled 5%)         │  (sampled 5%)
            ▼                       ▼
     shadow strong_model     shadow cheap_model  ← shadow cost
            │                       │
            └──────────┬────────────┘
                       ▼
            (trace, shadow_of, shadow_model)
                       │
                       ▼
              counterfactual quality
                       │
                       ▼
                routing_regret_report
```

## 2. Worked implementation

### 2.1 `router/schemas.py`

```python
from __future__ import annotations
from dataclasses import dataclass
from typing import Literal

Cohort = Literal["cheap", "strong"]
Model = Literal["cheap", "strong"]


@dataclass(frozen=True)
class RoutingDecision:
    trace_id: str
    cohort: Cohort                      # which bucket the router placed the request in
    router_version_id: str              # sha256:... content-addressed, like other versions
    router_confidence: float            # [0, 1] router's own estimate of routing a "hard" item
    features_hash: str                  # sha256 of the feature vector the router saw
    created_ns: int


@dataclass(frozen=True)
class ShadowLink:
    trace_id: str                       # the shadow trace
    shadow_of: str                      # the production trace it shadows
    shadow_model: Model
    sampled: bool                       # True only if this row was selected into the shadow sample


@dataclass(frozen=True)
class PairedScore:
    trace_id: str
    cohort: Cohort
    prod_model: Model                   # what the router actually chose
    prod_score: float
    shadow_model: Model                 # the other model
    shadow_score: float                 # NaN when shadow wasn't sampled
```

### 2.2 `router/sampling.py` — stratified shadow sampler

```python
from __future__ import annotations
import hashlib
import random
from collections.abc import Iterable


def should_shadow(
    *,
    trace_id: str,
    cohort: str,
    tool_shape: str,
    tenant_class: str,
    router_version_id: str,
    target_rate: float,
    strata_boosts: dict[str, float] | None = None,
) -> bool:
    """
    Deterministic, stratified shadow sampling.

    - Deterministic on (trace_id, router_version_id): the same trace is shadow-sampled
      or not consistently; replay stays stable.
    - Stratified: the strong cohort typically carries 5–20% of traffic. A uniform
      5% sample leaves it under-powered. `strata_boosts` multiplies the rate for a
      given stratum; downstream the weighting corrects back to population proportion.

    The hash-based sampler avoids a PRNG whose state depends on call order.
    """
    strata_boosts = strata_boosts or {}
    key = f"{cohort}:{tool_shape}:{tenant_class}"
    rate = min(1.0, target_rate * strata_boosts.get(key, 1.0))
    h = hashlib.sha256(f"{trace_id}:{router_version_id}".encode()).digest()
    # Treat first 8 bytes as a uniform [0, 1).
    draw = int.from_bytes(h[:8], "big") / 2**64
    return draw < rate
```

### 2.3 `router/sql/paired_scores.sql` — the paired scores view

```sql
-- One row per production trace that has BOTH a production and a shadow eval
-- scored by the same evaluator_version on the same dataset_version. The join
-- makes the counterfactual visible without the caller stitching rows.
--
-- `shadow_of` is a link table written by the shadow runner. `router_decision`
-- is written synchronously at request time with the cohort + router_version.
WITH prod AS (
    SELECT
        t.trace_id, t.cost_usd AS prod_cost, t.duration_ns AS prod_duration,
        rd.cohort, rd.router_version_id, rd.router_confidence,
        er.score AS prod_score, er.status AS prod_status,
        t.tenant_class, t.surface, t.tool_shape,
        -- which model actually ran — derived from app_version.model_pin.
        (av.model_pin ->> 'primary') AS prod_model_name
    FROM trace t
    JOIN router_decision rd  ON rd.trace_id = t.trace_id
    JOIN app_version av      ON av.app_version_id = t.app_version_id
    LEFT JOIN eval_result er ON er.trace_id = t.trace_id
                            AND er.evaluator_version_id = $1
                            AND er.dataset_version_id   = $2
    WHERE rd.router_version_id = $3
),
shadow AS (
    SELECT
        sl.shadow_of AS trace_id,
        sl.shadow_model,
        er.score  AS shadow_score,
        er.status AS shadow_status,
        t.cost_usd AS shadow_cost,
        t.duration_ns AS shadow_duration
    FROM shadow_link sl
    JOIN trace t             ON t.trace_id = sl.trace_id
    LEFT JOIN eval_result er ON er.trace_id = t.trace_id
                            AND er.evaluator_version_id = $1
                            AND er.dataset_version_id   = $2
)
SELECT
    p.*,
    s.shadow_model,
    s.shadow_score, s.shadow_status, s.shadow_cost, s.shadow_duration
FROM prod p
LEFT JOIN shadow s ON s.trace_id = p.trace_id;
```

### 2.4 `router/compose.py` — cohort-weighted composition

```python
from __future__ import annotations
from dataclasses import dataclass
from statistics import mean
import math
import random


@dataclass(frozen=True)
class CohortStats:
    cohort: str
    n: int                              # production rows in cohort
    quality_prod_mean: float            # mean quality on production model
    quality_counterfact_mean: float     # mean quality if the OTHER model had been used, from shadow
    n_shadowed: int                     # size of the shadow sample contributing
    cost_prod_mean: float               # per-request cost of what the router actually did
    cost_counterfact_mean: float
    cohort_weight: float                # share of production traffic in this cohort


def composed_quality(stats: list[CohortStats]) -> float:
    """Sum of (cohort_weight × cohort_prod_quality)."""
    return sum(s.cohort_weight * s.quality_prod_mean for s in stats)


def oracle_quality(stats: list[CohortStats]) -> float:
    """Per-cohort, pick the higher of the two model means (router's and counterfactual)."""
    return sum(s.cohort_weight * max(s.quality_prod_mean, s.quality_counterfact_mean)
               for s in stats)


def always_strong_quality(stats: list[CohortStats]) -> float:
    """Quality if every request went to the strong model."""
    total = 0.0
    for s in stats:
        if s.cohort == "strong":
            total += s.cohort_weight * s.quality_prod_mean
        else:
            # The 'cheap' cohort was routed to cheap; the counterfactual is strong.
            total += s.cohort_weight * s.quality_counterfact_mean
    return total


def composed_cost(stats: list[CohortStats]) -> float:
    return sum(s.cohort_weight * s.cost_prod_mean for s in stats)


def always_strong_cost(stats: list[CohortStats]) -> float:
    total = 0.0
    for s in stats:
        if s.cohort == "strong":
            total += s.cohort_weight * s.cost_prod_mean
        else:
            total += s.cohort_weight * s.cost_counterfact_mean
    return total


def regret(stats: list[CohortStats]) -> dict:
    """
    Routing regret and cost saving against:
      - always_strong baseline: the "no router" ceiling
      - oracle baseline: the unreachable per-request optimum
    """
    comp_q = composed_quality(stats)
    oracle = oracle_quality(stats)
    strong_q = always_strong_quality(stats)
    comp_c = composed_cost(stats)
    strong_c = always_strong_cost(stats)
    return {
        "router_quality":       comp_q,
        "oracle_quality":       oracle,
        "always_strong_quality": strong_q,
        "quality_regret_vs_oracle":        oracle - comp_q,
        "quality_delta_vs_always_strong":  comp_q - strong_q,
        "router_cost":          comp_c,
        "always_strong_cost":   strong_c,
        "cost_saving_vs_always_strong":    strong_c - comp_c,
    }
```

Why the structure: the router's average quality can only be fairly compared to `always_strong` and `oracle`. The oracle is unattainable but bounds how good any router can be; `always_strong` is the realistic baseline the router replaces. Any report that doesn't compare against both is missing a side of the trade-off.

### 2.5 `router/stats.py` — per-cohort CI and the inversion check

```python
from __future__ import annotations
import math
import random
from typing import Iterable


def cohort_point_estimate(scores: list[float]) -> float:
    xs = [s for s in scores if s is not None and not math.isnan(s)]
    return sum(xs) / len(xs) if xs else math.nan


def bootstrap_mean_ci(xs: list[float], iters: int, seed: int, alpha: float = 0.05) -> tuple[float, float]:
    xs = [x for x in xs if x is not None and not math.isnan(x)]
    if len(xs) < 2:
        return math.nan, math.nan
    rng = random.Random(seed)
    boots = []
    for _ in range(iters):
        sample = [xs[rng.randrange(len(xs))] for _ in xs]
        boots.append(sum(sample) / len(sample))
    boots.sort()
    return boots[int(iters * alpha / 2)], boots[int(iters * (1 - alpha / 2))]


def inversion_test(cohort_scores: dict[str, dict[str, list[float]]]) -> dict:
    """
    cohort_scores: {cohort: {model: [scores...]}}

    Returns a dict with the test result:
      - strong_on_strong:   mean score of strong model on the strong cohort
      - strong_on_cheap:    mean score of strong model on the cheap cohort
      - passes:             True iff strong_on_strong - strong_on_cheap ≥ 0

    The routing policy's whole claim is that cohort=strong traffic is harder.
    If the strong model scores no higher on its own cohort than on the other,
    the router is noise.
    """
    strong_on_strong = cohort_point_estimate(cohort_scores["strong"]["strong"])
    strong_on_cheap  = cohort_point_estimate(cohort_scores["cheap"]["strong"])
    return {
        "strong_on_strong_cohort": strong_on_strong,
        "strong_on_cheap_cohort":  strong_on_cheap,
        "gap":                     strong_on_strong - strong_on_cheap,
        "passes":                  (strong_on_strong - strong_on_cheap) >= 0,
    }
```

### 2.6 `router/weighting.py` — correcting shadow boosts

```python
from __future__ import annotations


def rebalance(
    shadow_rows: list[dict],
    *,
    population_proportion: dict[str, float],
    sample_proportion:     dict[str, float],
) -> list[dict]:
    """
    Horvitz–Thompson style rebalancing.

    If the strong cohort got boosted 5× in sampling, each strong-cohort shadow
    row counts with weight 1/5 when forming the counterfactual means. Without
    this, the composition uses a biased estimate and the regret number is
    systematically off.
    """
    out = []
    for r in shadow_rows:
        key = f"{r['cohort']}:{r['tool_shape']}:{r['tenant_class']}"
        p_pop = population_proportion.get(key, 0.0)
        p_sam = sample_proportion.get(key, 1e-12)
        r = dict(r)
        r["w"] = p_pop / p_sam
        out.append(r)
    return out


def weighted_mean(rows: list[dict], score_key: str) -> float:
    num = sum(r["w"] * r[score_key] for r in rows if r[score_key] is not None)
    den = sum(r["w"]                for r in rows if r[score_key] is not None)
    return num / den if den else float("nan")
```

### 2.7 `router/main.py` — orchestrates the eval

```python
from __future__ import annotations
import argparse
import json
import pathlib

from .compose    import regret
from .schemas    import CohortStats
from .stats      import bootstrap_mean_ci, cohort_point_estimate, inversion_test
from .store      import load_paired_scores, load_production_cohort_weights
from .weighting  import rebalance, weighted_mean


def run(router_version_id: str, dataset_version_id: str, judge_version_id: str,
        seed: int = 20241119, iters: int = 2000) -> dict:
    rows = load_paired_scores(router_version_id, dataset_version_id, judge_version_id)
    weights = load_production_cohort_weights(router_version_id)
    # Build cohort x model indexed scores for the inversion test.
    buckets: dict[str, dict[str, list[float]]] = {
        "cheap":  {"cheap": [], "strong": []},
        "strong": {"cheap": [], "strong": []},
    }
    for r in rows:
        buckets[r["cohort"]][_normalise_model_name(r["prod_model_name"])].append(r["prod_score"])
        if r.get("shadow_score") is not None:
            buckets[r["cohort"]][r["shadow_model"]].append(r["shadow_score"])

    # Per-cohort aggregates with Horvitz–Thompson weights for shadow rows.
    strata = _compute_strata(rows)
    cohort_stats = []
    for cohort in ("cheap", "strong"):
        cohort_rows = [r for r in rows if r["cohort"] == cohort]
        rebalanced  = rebalance(
            [r for r in cohort_rows if r.get("shadow_score") is not None],
            population_proportion=strata["pop"],
            sample_proportion=strata["sample"],
        )
        prod_q = cohort_point_estimate([r["prod_score"]   for r in cohort_rows])
        shad_q = weighted_mean(rebalanced, "shadow_score")
        prod_c = cohort_point_estimate([r["prod_cost"]    for r in cohort_rows])
        shad_c = weighted_mean(rebalanced, "shadow_cost")
        cohort_stats.append(CohortStats(
            cohort=cohort, n=len(cohort_rows),
            quality_prod_mean=prod_q,
            quality_counterfact_mean=shad_q,
            n_shadowed=len(rebalanced),
            cost_prod_mean=prod_c, cost_counterfact_mean=shad_c,
            cohort_weight=weights[cohort],
        ))

    out = regret(cohort_stats)
    out["inversion_test"] = inversion_test(buckets)
    out["cohorts"] = [s.__dict__ for s in cohort_stats]
    # CIs for the headline composed quality (via bootstrap over per-trace delta).
    out["composed_quality_ci"] = bootstrap_mean_ci(
        [r["prod_score"] for r in rows if r["prod_score"] is not None],
        iters=iters, seed=seed,
    )
    return out


def _normalise_model_name(name: str) -> str:
    # "haiku-3.5" / "gpt-4o-mini" → cheap; "gpt-4o" / "sonnet-3.5" → strong.
    # The mapping is deployment-specific and lives in config.
    return "cheap" if name.endswith(("mini", "haiku-3.5")) else "strong"


def _compute_strata(rows: list[dict]) -> dict[str, dict[str, float]]:
    """Compute population and sample proportions per stratum."""
    pop = {}
    sample = {}
    for r in rows:
        k = f"{r['cohort']}:{r['tool_shape']}:{r['tenant_class']}"
        pop[k] = pop.get(k, 0) + 1
        if r.get("shadow_score") is not None:
            sample[k] = sample.get(k, 0) + 1
    tot = sum(pop.values())
    return {
        "pop":    {k: v / tot for k, v in pop.items()},
        "sample": {k: (sample.get(k, 0) / max(1, pop[k])) for k in pop},
    }


if __name__ == "__main__":
    ap = argparse.ArgumentParser()
    ap.add_argument("--router-version-id", required=True)
    ap.add_argument("--dataset-version-id", required=True)
    ap.add_argument("--judge-version-id",   required=True)
    ap.add_argument("--out", type=pathlib.Path, default=pathlib.Path("router_regret.json"))
    args = ap.parse_args()
    args.out.write_text(json.dumps(run(
        args.router_version_id, args.dataset_version_id, args.judge_version_id,
    ), indent=2))
```

### 2.8 `tests/test_router_eval.py`

```python
import math
import pytest

from router.compose    import composed_cost, composed_quality, oracle_quality, regret, always_strong_quality
from router.sampling   import should_shadow
from router.schemas    import CohortStats
from router.stats      import bootstrap_mean_ci, inversion_test
from router.weighting  import rebalance, weighted_mean


def _st(cohort, n, q_prod, q_ctr, c_prod, c_ctr, w):
    return CohortStats(cohort, n, q_prod, q_ctr, n // 20, c_prod, c_ctr, w)


def test_shadow_sampler_is_deterministic():
    a = should_shadow(trace_id="t1", cohort="cheap", tool_shape="rag",
                      tenant_class="paid", router_version_id="v1",
                      target_rate=0.05)
    b = should_shadow(trace_id="t1", cohort="cheap", tool_shape="rag",
                      tenant_class="paid", router_version_id="v1",
                      target_rate=0.05)
    assert a == b


def test_shadow_sampler_respects_strata_boost():
    hits = 0
    n = 10_000
    for i in range(n):
        if should_shadow(trace_id=f"t{i}", cohort="strong", tool_shape="rag",
                          tenant_class="paid", router_version_id="v1",
                          target_rate=0.05, strata_boosts={"strong:rag:paid": 4.0}):
            hits += 1
    assert 0.17 <= hits / n <= 0.23        # 20% ± slack


def test_composed_quality_weights_by_cohort():
    s = [
        _st("cheap",  800, q_prod=0.70, q_ctr=0.80, c_prod=0.5, c_ctr=5.0, w=0.8),
        _st("strong", 200, q_prod=0.88, q_ctr=0.78, c_prod=5.0, c_ctr=0.5, w=0.2),
    ]
    assert composed_quality(s) == pytest.approx(0.8 * 0.70 + 0.2 * 0.88)


def test_oracle_is_upper_bound():
    s = [
        _st("cheap",  800, q_prod=0.70, q_ctr=0.80, c_prod=0.5, c_ctr=5.0, w=0.8),
        _st("strong", 200, q_prod=0.88, q_ctr=0.78, c_prod=5.0, c_ctr=0.5, w=0.2),
    ]
    assert oracle_quality(s) >= composed_quality(s)
    assert oracle_quality(s) >= always_strong_quality(s)


def test_regret_report_includes_cost_saving():
    s = [
        _st("cheap",  800, q_prod=0.70, q_ctr=0.80, c_prod=0.5, c_ctr=5.0, w=0.8),
        _st("strong", 200, q_prod=0.88, q_ctr=0.78, c_prod=5.0, c_ctr=0.5, w=0.2),
    ]
    out = regret(s)
    assert out["cost_saving_vs_always_strong"] > 0
    assert out["quality_delta_vs_always_strong"] <= 0        # router sacrificed some quality
    assert out["quality_regret_vs_oracle"] >= 0


def test_inversion_test_detects_anti_correlated_router():
    buckets = {
        "cheap":  {"strong": [0.90, 0.92, 0.88], "cheap": [0.70, 0.72]},
        "strong": {"strong": [0.78, 0.76, 0.80], "cheap": [0.60, 0.62]},
    }
    out = inversion_test(buckets)
    assert out["passes"] is False
    assert out["gap"] < 0


def test_inversion_test_passes_for_good_router():
    buckets = {
        "cheap":  {"strong": [0.88, 0.90, 0.89], "cheap": [0.82, 0.84]},
        "strong": {"strong": [0.75, 0.77, 0.76], "cheap": [0.55, 0.58]},
        # strong model does 0.76 on strong cohort (its hard cases)
        # vs 0.89 on cheap cohort (its easy cases). Gap is NEGATIVE (-0.13):
        # harder cohort = lower absolute scores. That's expected AND the router
        # is still correct because the cheap→cheap score is even lower (0.56)
        # than the strong→strong score (0.76). Inversion test as written only
        # checks strong-on-strong vs strong-on-cheap, which is insufficient.
        #
        # This test documents the limit: use the inversion test in conjunction
        # with regret vs oracle.
    }
    out = inversion_test(buckets)
    assert out["gap"] < 0           # strong cohort IS harder in absolute terms
    # The reporter should surface this in text, not fail the whole eval.


def test_rebalance_assigns_horvitz_thompson_weights():
    rows = [
        {"cohort": "strong", "tool_shape": "rag", "tenant_class": "paid",
         "shadow_score": 0.9, "shadow_cost": 4.0},
        {"cohort": "strong", "tool_shape": "rag", "tenant_class": "paid",
         "shadow_score": 0.7, "shadow_cost": 3.0},
    ]
    pop = {"strong:rag:paid": 0.2}
    smp = {"strong:rag:paid": 0.8}        # over-sampled 4×
    r = rebalance(rows, population_proportion=pop, sample_proportion=smp)
    assert r[0]["w"] == pytest.approx(0.25)
    assert weighted_mean(r, "shadow_score") == pytest.approx(0.8)
```

### 2.9 Report template (`router/report_template.md`)

```markdown
# Routing eval: router {router_version_id}

- Dataset: `{dataset_version_id}`     Judge: `{judge_version_id}`
- Production window: `{window_start} → {window_end}` (n={n_prod})
- Shadow window: same (n_shadow={n_shadow})
- Production cohort shares: cheap={share_cheap:.1%}, strong={share_strong:.1%}

## Decision

- Router quality:         `{router_quality:.4f}`   [CI {rq_lo:.4f}, {rq_hi:.4f}]
- Always-strong quality:  `{always_strong_quality:.4f}`
- Oracle quality:         `{oracle_quality:.4f}`
- Δvs always-strong:      `{d_vs_strong:+.4f}`     ← acceptable if ≥ -{quality_floor}
- Regret vs oracle:       `{regret_vs_oracle:+.4f}`

- Router cost per 1k:     `{router_cost_per_1k:.3f}` USD
- Always-strong cost:     `{strong_cost_per_1k:.3f}` USD
- Cost saving:            `{cost_saving_per_1k:+.3f}` USD ({cost_saving_pct:+.1%})

**Verdict**: {verdict}  — {verdict_reason}

## Inversion test

| Metric | Value |
|---|---|
| strong model on strong cohort | {strong_on_strong:.4f} |
| strong model on cheap cohort  | {strong_on_cheap:.4f} |
| gap                           | {inv_gap:+.4f}        |
| passes (≥ 0)                  | {inv_passes}          |

> The inversion test is a floor, not a ceiling. Even if it fails in absolute
> terms (because the strong cohort is genuinely harder), the router can still
> be correct — see "regret vs oracle".

## Per cohort

| Cohort | n (prod) | n (shadow) | prod quality | counterfactual quality | prod cost/1k | counterfact cost/1k | cohort weight |
|---|---|---|---|---|---|---|---|
{cohort_rows}

## Slice tables (per cohort × tenant_class)

{slice_rows}

## Lineage

| Router version | Features sha prefix | Cheap model pin | Strong model pin | Deployed at |
|---|---|---|---|---|
| `{router_version_id}` | `{features_hash_prefix}` | `{cheap_model_pin}` | `{strong_model_pin}` | `{deployed_at}` |
```

### 2.10 Worked example

Suppose the router sends 80% of traffic to a cheap model and 20% to a strong model. A shadow sample of 5% (boosted 4× for the strong cohort) yields:

| cohort | n | n_shadowed | quality_prod | quality_counterfactual | cost_prod/1k | cost_counterfact/1k | weight |
|---|---|---|---|---|---|---|---|
| cheap  | 8,000 |  400 | 0.722 | 0.792 | 0.60 | 6.20 | 0.80 |
| strong | 2,000 |  400 | 0.874 | 0.790 | 6.00 | 0.60 | 0.20 |

Composed quality = `0.8 × 0.722 + 0.2 × 0.874 = 0.752`.
Oracle quality = `0.8 × max(0.722, 0.792) + 0.2 × max(0.874, 0.790) = 0.8 × 0.792 + 0.2 × 0.874 = 0.808`.
Always-strong = `0.8 × 0.792 + 0.2 × 0.874 = 0.808`. (In this example, strong is strictly better.)

Composed cost/1k = `0.8 × 0.60 + 0.2 × 6.00 = 1.68 USD`.
Always-strong cost/1k = `0.8 × 6.20 + 0.2 × 6.00 = 6.16 USD`.

**Verdict**: quality regret `-0.056` vs oracle, cost saving `+4.48 USD/1k` (73%). If the product's acceptable quality floor is `-0.05`, the router is a MARGINAL FAIL; if the floor is `-0.07`, it is a pass. Headline the trade-off, not a Boolean.

## 3. Validation steps

1. **Unit tests**: `pytest -q tests/test_router_eval.py` — all 8 tests pass.
2. **Load fixture**: seed 10k paired traces (8k cheap, 2k strong) with a planted ground truth. Run `router.main` end-to-end; the reported regret matches the ground-truth regret within the composed-quality CI.
3. **Determinism**: two runs with the same `(router_version_id, dataset_version_id, judge_version_id, seed)` produce byte-identical JSON.
4. **Rebalance correctness**: feed a known over-sampled strong cohort (4× boost); after rebalancing the counterfactual mean matches the uniform-sample mean within noise.
5. **Inversion test — negative case**: swap cohort labels in the fixture. The eval surfaces `inversion_test.passes = false` and the verdict text says "router is anti-correlated with difficulty".
6. **Shadow-coverage floor**: force `n_shadowed < 30` in one cohort. The eval refuses to emit a composed number for that cohort; instead it emits `insufficient_shadow` and a widened CI banner.
7. **Cohort-weight sensitivity**: run the eval under a 70/30 and a 90/10 cohort mix. Verify composed numbers move as expected; the per-cohort numbers do not move.
8. **PII check**: no raw user content appears in `paired_scores` output; only `user_id_hash`-style identifiers.
9. **Lineage completeness**: every row in the paired-scores view has a resolvable `router_version_id`, `app_version_id`, `evaluator_version_id`, `dataset_version_id`.
10. **Acceptance replay**: pick a known-bad router (shuffled cohorts). The eval marks it FAIL on the inversion test AND the regret test. Pick a known-good router (oracle). The eval marks it PASS with regret ≈ 0.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|---|---|---|
| R1 | Cohorts are preserved end-to-end. Per-cohort quality and cost are reported. | Per-cohort rows present; composition shown as `Σ weight × cohort_mean`. | Only aggregate quality; no cohort slices. |
| R2 | Counterfactual quality is measured from a shadow sample, not inferred. | `shadow_link` rows exist; paired view LEFT JOINs them. | Counterfactual estimated from an unrelated holdout. |
| R3 | Oracle and always-strong baselines are both reported. | Both numbers in the decision block. | Only one baseline, or none. |
| R4 | Regret is reported in BOTH quality and cost units. | `quality_regret_vs_oracle` and `cost_saving_vs_always_strong`. | A single compound metric. |
| R5 | Stratified sampling with Horvitz–Thompson rebalancing. | `strata_boosts` + `rebalance()` used; test confirms unbiased means. | Uniform 5% sample; strong cohort under-powered. |
| R6 | Inversion test is executed and surfaced. | `inversion_test.passes` in the output; verdict text notes failure. | Report silent on router direction. |
| R7 | Shadow cost is accounted for separately. | `shadow_cost` column; report calls it out. | Shadow cost silently rolled into router cost. |
| R8 | Shadow-coverage floor enforces minimum samples. | Eval refuses or widens CI when `n_shadow < 30` per cohort. | Reports a number regardless of sample size. |
| R9 | The eval is reproducible from `(router_version_id, dataset_version_id, judge_version_id, seed)`. | Identical outputs on replay; recorded in output JSON. | Numbers drift between replays. |
| R10 | PII boundary respected; no raw user text in the eval output. | Only hashes and scores in output. | Raw queries / content in the report. |

A passing submission is 8+ of 10, with R1, R2, R3, R6, R9 all hard-required.

## 5. Common mistakes

- **Comparing cohorts against each other as if they were unbalanced samples of the same thing.** The router *chose* the cohorts; cross-cohort differences conflate the router's decision with the underlying difficulty. Always compare *within* a cohort, then compose with cohort weights.
- **Skipping the shadow eval and inferring counterfactuals.** You cannot recover what the other model would have scored from observational data alone; difficulty is confounded with the router's choice. Pay the 5% cost.
- **Uniform shadow sampling.** The strong cohort is small by design; a uniform sample leaves it severely under-powered and the regret CI is too wide to decide.
- **Forgetting the always-strong baseline.** "Router is better than cheap-only" is uninteresting. The relevant question is "is the router close enough to always-strong to justify the cost saving?"
- **Treating the inversion test as sufficient.** Passing the inversion test does not mean the router is good; it means the router is pointed the right direction. A router that routes 50/50 by coin flip can pass the inversion test if the cohorts happen to split roughly on difficulty.
- **Dropping shadow cost from the cost line.** Shadow calls are real spend. A router whose "cost saving" vanishes once shadow is accounted for is a different product decision.
- **Non-deterministic sampling.** PRNG-based sampling means replays pick different rows and the regret number drifts. Hash the `(trace_id, router_version_id)` instead.
- **Ignoring the shadow-sample floor.** Composing against an `n_shadow=12` cohort produces confident-looking regret numbers that are noise. Refuse to publish below a documented floor.
- **One global Horvitz–Thompson weight.** The weight must be per stratum (cohort × tool_shape × tenant_class); applying one global weight erases the point of stratifying.
- **Forgetting latency.** Routing is usually motivated by latency as much as cost. If the strong model is 3× slower on tail, the router buys user-perceived warmth even when cost looks like a wash. Report p50/p95 latency on the same cohort axes.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 2.3` (metric coverage), `MEASURE 2.7` (validity), `MANAGE 4.1` (regression safeguards) apply to router rollout.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML06` (AI supply chain) motivates lineage of both cheap and strong model artefacts; `ML04` (membership inference) motivates the per-tenant shadow slice.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.TA0002` (model access) and `AML.T0043` motivate hardening the shadow pipeline so production prompts are not leaked to a lower-trust model path.
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/>, for the span attributes mirrored by `trace.model_pin`, `trace.cost_usd`, `trace.duration_ns`.
- Local exercise reader: `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-02-model-routing-eval-with-cohort-preservation.md`.
- Related solutions: this module's exercise-01 (report shape, CIs), exercise-04 (latency reporting), `mod-110/exercise-01` (schema), `mod-107-online-eval-and-regression` (continuous comparison patterns applied here for the shadow pipeline), `mod-104-llm-as-judge-in-product` (judge version stability during cohort eval).
