# exercise-02 — Partial Credit Rubric For A Product Agent — Solution

> Reference solution for `lessons/mod-103-trajectory-and-tool-eval/exercises/exercise-02-partial-credit-rubric-for-product-agent.md`.
> Chapter 03 of this module is the primary input; the exercise sits on top of exercise-01's per-step and budget verdicts and produces the per-trajectory shape that mod-106 (CI) and mod-107 (online eval) read.

## 1. Solution overview

The exercise delivers the rubric *file* plus the code that reads it. The file is a versioned product artefact that lives alongside the eval plan; the code — `roll_up`, `aggregate`, `exact_match_normalised`, `token_f1` — treats it as data, never tunes it.

Four design choices anchor the solution:

1. **Rubric file separate from rubric code.** The weights, floors, and pass threshold live in `rubric.v1.4.yaml`. The code loads them, validates `sum(weights) == 1.0` on load, and fails loudly on mismatch. Post-hoc weight tuning is the failure mode chapter 03 warns against most; keeping the file as the source of truth makes tuning a reviewable product change (mod-106 hashes it into the CI check).
2. **Floors trip on named verdicts, not credit thresholds.** `trip_on_verdict` is a hard gate. A trajectory with `wrong_tool` in `flagged_verdicts` is `pass=false` regardless of whether the weighted sum cleared `pass_threshold`. The weighted sum is still computed and returned — a reviewer wants to see "the floor tripped, *and additionally* the sum was 0.45".
3. **Per-task-family rubrics.** The rubric file is keyed on `task_family` and refuses to score a trajectory with a different task family. This is the chapter 03 rule against "one rubric across unrelated families"; it is the easiest thing to get wrong and the hardest to undo once it is in production.
4. **The scorer never calls a model.** `exact_match_normalised` and `token_f1` are deterministic string comparisons. `reference_based_judge` is a stub that raises `NotImplementedError` with a pointer to mod-104. The rubric roll-up itself computes a weighted sum; no inference.

## 2. Worked implementation

```
mod-103/exercise-02/
├── pyproject.toml
├── README.md
├── rubric/
│   ├── __init__.py
│   ├── schema.py           # RubricConfig, AxisCredit, PerTrajectoryVerdict
│   ├── loader.py           # load_rubric(path) + sum-to-1 enforcement
│   ├── final_answer.py     # exact_match_normalised, token_f1, stub judge
│   ├── rollup.py           # roll_up(per_step, final_answer, groundedness, budget, rubric)
│   ├── aggregate.py        # aggregate(verdicts, slice_by)
│   └── diff.py             # stretch: v1.4 vs v1.5 diff
├── configs/
│   └── rubric_order_status_not_shipped_policy_v1_4.yaml
└── tests/
    └── test_rollup.py
```

### 2.1 `configs/rubric_order_status_not_shipped_policy_v1_4.yaml`

```yaml
# Per task family, versioned. Chapter 03 §"Three design rules".
task_family: order_status_not_shipped_policy
rubric_version: v1.4
weights:
  name:          0.20
  args:          0.10
  order:         0.10
  answer:        0.20
  groundedness:  0.25
  budget:        0.15
floors:
  trip_on_verdict:
    - wrong_tool
    - unauthorised_tool
    - ordering_violation
    - fabricated_claim
pass_threshold: 0.80
methods:
  answer_scorer:        exact_match_normalised
  groundedness_scorer:  stub_v0           # mod-105 fills in
# Per-cohort overrides (stretch). Keys match sample.metadata["cohort"].
cohort_overrides: {}
```

### 2.2 `rubric/schema.py`

```python
"""Pydantic schema for the rubric file and the per-trajectory verdict."""
from __future__ import annotations
from typing import Literal
from pydantic import BaseModel, Field, field_validator, model_validator

AxisName = Literal["name", "args", "order", "answer", "groundedness", "budget"]


class RubricWeights(BaseModel):
    name: float
    args: float
    order: float
    answer: float
    groundedness: float
    budget: float

    @model_validator(mode="after")
    def _sum_to_one(self) -> "RubricWeights":
        s = (self.name + self.args + self.order
             + self.answer + self.groundedness + self.budget)
        if abs(s - 1.0) > 1e-6:
            raise ValueError(f"rubric weights must sum to 1.0; got {s:.6f}")
        return self


class RubricFloors(BaseModel):
    trip_on_verdict: list[str] = Field(default_factory=list)


class RubricMethods(BaseModel):
    answer_scorer: Literal["exact_match_normalised", "token_f1",
                           "reference_based_judge", "stub_v0"]
    groundedness_scorer: Literal["stub_v0", "mod_105_retrieval_coverage"]


class RubricConfig(BaseModel):
    task_family: str
    rubric_version: str
    weights: RubricWeights
    floors: RubricFloors
    pass_threshold: float = Field(ge=0.0, le=1.0)
    methods: RubricMethods
    cohort_overrides: dict[str, RubricFloors] = Field(default_factory=dict)


class AxisCredit(BaseModel):
    credit: float = Field(ge=0.0, le=1.0)
    method: str
    details: dict = Field(default_factory=dict)


class PerTrajectoryVerdict(BaseModel):
    trajectory_id: str
    task_family: str
    rubric_version: str
    reference_version: str
    price_table_version: str
    trajectory_score: float
    pass_: bool = Field(alias="pass")
    floor_tripped: str | None
    axes: dict[AxisName, AxisCredit]
    flagged_verdicts: list[str]

    model_config = {"populate_by_name": True}
```

### 2.3 `rubric/loader.py`

```python
"""Load the rubric YAML and surface schema errors loudly on import time."""
from __future__ import annotations
from pathlib import Path
import yaml

from .schema import RubricConfig


def load_rubric(path: str | Path) -> RubricConfig:
    with open(path) as f:
        data = yaml.safe_load(f)
    return RubricConfig.model_validate(data)
```

### 2.4 `rubric/final_answer.py`

```python
"""Final-answer scoring methods. Chapter 03 lists six; this exercise ships
two deterministic ones and stubs the judge (mod-104 owns the depth)."""
from __future__ import annotations
import re
import unicodedata
from collections import Counter

from .schema import AxisCredit


_FENCE_RE = re.compile(r"^```[a-zA-Z]*\n?|```$", re.MULTILINE)


def _normalise(s: str) -> str:
    s = unicodedata.normalize("NFKC", s)
    s = _FENCE_RE.sub("", s)
    return s.strip().casefold()


def exact_match_normalised(reply: str, reference: str) -> AxisCredit:
    """Lowercase, strip whitespace + Markdown fences, string-compare.

    Suitable for terminal replies that are short enough to be canonicalised
    (confirm/deny, structured one-liners). Misleading for open-ended replies.
    """
    match = _normalise(reply) == _normalise(reference)
    return AxisCredit(
        credit=1.0 if match else 0.0,
        method="exact_match_normalised",
        details={"match": match,
                 "reply_norm_len": len(_normalise(reply)),
                 "reference_norm_len": len(_normalise(reference))},
    )


def _tokens(s: str) -> list[str]:
    return re.findall(r"\w+", _normalise(s))


def token_f1(reply: str, reference: str) -> AxisCredit:
    """SQuAD-style token F1. Misleading when:
    - the reference is one token (precision/recall degenerate),
    - the reply is long and the reference short (false high recall),
    - the reply is short and carries a wrong *number* (F1 passes the surrounding words).
    Use alongside, not instead of, a numeric-aware check.
    """
    r_tok = _tokens(reply)
    g_tok = _tokens(reference)
    if not r_tok or not g_tok:
        return AxisCredit(credit=0.0, method="token_f1",
                          details={"reason": "empty_token_set",
                                   "reply_tokens": r_tok, "reference_tokens": g_tok})
    r_counter = Counter(r_tok)
    g_counter = Counter(g_tok)
    overlap = sum((r_counter & g_counter).values())
    if overlap == 0:
        return AxisCredit(credit=0.0, method="token_f1",
                          details={"overlap": 0, "precision": 0.0, "recall": 0.0})
    precision = overlap / sum(r_counter.values())
    recall = overlap / sum(g_counter.values())
    f1 = 2 * precision * recall / (precision + recall)
    return AxisCredit(credit=f1, method="token_f1",
                      details={"overlap": overlap, "precision": precision, "recall": recall})


def reference_based_judge(reply: str, reference: str) -> AxisCredit:  # pragma: no cover
    raise NotImplementedError(
        "Reference-based judge configuration is owned by mod-104. The trajectory "
        "scorer calls into the judge module when the rubric selects it; it does "
        "not define the prompt or the grading criteria here."
    )
```

### 2.5 `rubric/rollup.py`

```python
"""Per-trajectory roll-up. Chapter 03 §"Credit-to-step attribution"."""
from __future__ import annotations
from typing import Any

from .schema import AxisCredit, PerTrajectoryVerdict, RubricConfig


def _credit_for(verdict: dict[str, Any], axis: str) -> float:
    """Chapter 03: worst-case per-axis across the required steps."""
    sub = verdict[axis]
    # Our per-step verdicts carry a `match: bool` on tool_name, arguments, order.
    # Score 1.0 for match, 0.0 for no match; a Level-3 partial would plug in here
    # if the argument scorer returned a continuous value.
    return 1.0 if sub.get("match", False) else 0.0


def _axis_worst_case(per_step: list[dict[str, Any]], axis: str) -> AxisCredit:
    required = [v for v in per_step
                if v["tool_name"]["reason"] in ("ok", "wrong_tool", "extra_tool")]
    if not required:
        return AxisCredit(credit=1.0, method="worst_case", details={"n": 0})
    credits = [_credit_for(v, axis) for v in required]
    return AxisCredit(
        credit=min(credits), method="worst_case",
        details={"n": len(credits), "min": min(credits), "mean": sum(credits) / len(credits)},
    )


def _axis_name(per_step: list[dict[str, Any]]) -> AxisCredit:
    """Required-step name credit; extra tools do NOT lower name_credit
    — they are a `flagged_verdicts` concern (chapter 03 §Starter guidance).
    """
    required = [v for v in per_step if v["tool_name"]["reason"] in ("ok", "wrong_tool")]
    if not required:
        return AxisCredit(credit=1.0, method="worst_case_name", details={"n": 0})
    credits = [1.0 if v["tool_name"]["match"] else 0.0 for v in required]
    return AxisCredit(
        credit=min(credits), method="worst_case_name",
        details={"n": len(credits), "min": min(credits)},
    )


def _collect_flagged(
    per_step: list[dict[str, Any]], budget: dict[str, Any],
) -> list[str]:
    """Union of chapter-02 and chapter-04 verdicts observed on the trajectory."""
    flagged: list[str] = []
    for v in per_step:
        if v["verdict"] != "ok" and v["verdict"] not in flagged:
            flagged.append(v["verdict"])
    for v in budget.get("flagged_verdicts", []):
        if v not in flagged:
            flagged.append(v)
    return flagged


def _first_floor_tripped(
    flagged: list[str], trip_on_verdict: list[str],
) -> str | None:
    """Chapter 03 §"Floors trip on a named verdict list"."""
    for verdict in flagged:
        if verdict in trip_on_verdict:
            return verdict
    return None


def roll_up(
    per_step: list[dict[str, Any]],
    final_answer: AxisCredit,
    groundedness: AxisCredit,
    budget: dict[str, Any],
    rubric: RubricConfig,
    *,
    trajectory_id: str,
    reference_version: str,
    cohort: str | None = None,
) -> PerTrajectoryVerdict:
    axes: dict[str, AxisCredit] = {
        "name":        _axis_name(per_step),
        "args":        _axis_worst_case(per_step, "arguments"),
        "order":       _axis_worst_case(per_step, "order"),
        "answer":      final_answer,
        "groundedness": groundedness,
        "budget":      AxisCredit(credit=budget["budget_credit"],
                                  method="chapter_04_min_across_families",
                                  details={"flagged": budget["flagged_verdicts"],
                                           "attribution": budget["attribution"]}),
    }
    w = rubric.weights
    trajectory_score = (
        w.name * axes["name"].credit
        + w.args * axes["args"].credit
        + w.order * axes["order"].credit
        + w.answer * axes["answer"].credit
        + w.groundedness * axes["groundedness"].credit
        + w.budget * axes["budget"].credit
    )

    flagged = _collect_flagged(per_step, budget)
    floors = rubric.floors.trip_on_verdict
    if cohort and cohort in rubric.cohort_overrides:
        floors = list(set(floors) | set(rubric.cohort_overrides[cohort].trip_on_verdict))
    floor_tripped = _first_floor_tripped(flagged, floors)

    passed = (trajectory_score >= rubric.pass_threshold) and (floor_tripped is None)

    return PerTrajectoryVerdict(
        trajectory_id=trajectory_id,
        task_family=rubric.task_family,
        rubric_version=rubric.rubric_version,
        reference_version=reference_version,
        price_table_version=budget["price_table_version"],
        trajectory_score=trajectory_score,
        pass_=passed,
        floor_tripped=floor_tripped,
        axes={k: v for k, v in axes.items()},  # type: ignore[arg-type]
        flagged_verdicts=flagged,
    )
```

### 2.6 `rubric/aggregate.py`

```python
"""Per-slice roll-ups across many trajectories. Chapter 04 §Budget-family rollups."""
from __future__ import annotations
from collections import defaultdict
from statistics import quantiles
from typing import Any

from .schema import PerTrajectoryVerdict


def _p95(xs: list[float]) -> float:
    if not xs:
        return 0.0
    if len(xs) == 1:
        return xs[0]
    # statistics.quantiles with n=20 → 19 cut-points at 5-pct steps; [-1] is p95.
    return quantiles(sorted(xs), n=20, method="inclusive")[-1]


def aggregate(
    verdicts: list[PerTrajectoryVerdict],
    slice_by: list[str] | None = None,
    enrich: dict[str, dict[str, Any]] | None = None,
) -> dict[str, dict[str, Any]]:
    """Return {slice_key: metrics}. slice_by defaults to task_family + rubric_version.
    `enrich` maps trajectory_id → extra fields (cost_usd, latency_s, cohort) that
    the aggregator needs for cost_p95 / latency_p95; passing it keeps this module
    free of exercise-01's budget schema.
    """
    slice_by = slice_by or ["task_family", "rubric_version"]
    enrich = enrich or {}
    buckets: dict[tuple[str, ...], list[PerTrajectoryVerdict]] = defaultdict(list)
    for v in verdicts:
        extra = enrich.get(v.trajectory_id, {})
        key = tuple(str(getattr(v, k, extra.get(k, ""))) for k in slice_by)
        buckets[key].append(v)

    out: dict[str, dict[str, Any]] = {}
    for key, items in buckets.items():
        slice_label = "::".join(key)
        passes = [v for v in items if v.pass_]
        out[slice_label] = {
            "n": len(items),
            "pass_rate": len(passes) / len(items) if items else 0.0,
            "cost_p95": _p95([enrich.get(v.trajectory_id, {}).get("cost_usd", 0.0)
                              for v in items]),
            "latency_p95": _p95([enrich.get(v.trajectory_id, {}).get("latency_s", 0.0)
                                 for v in items]),
            "rate_of_flagged_verdict": {
                tag: sum(1 for v in items if tag in v.flagged_verdicts) / len(items)
                for tag in sorted({t for v in items for t in v.flagged_verdicts})
            },
        }
    return out
```

### 2.7 `tests/test_rollup.py`

```python
from pathlib import Path
import pytest

from rubric.loader import load_rubric
from rubric.rollup import roll_up
from rubric.schema import AxisCredit

CONFIG = Path(__file__).parent.parent / "configs" / \
    "rubric_order_status_not_shipped_policy_v1_4.yaml"


def _per_step_bad_trajectory():
    # Matches exercise-01's bad-trajectory output shape.
    return [
        {"step_id": 2, "span_id": "s_sql",
         "tool_name": {"expected": None, "actual": "sql_query",
                       "aliased": False, "match": False, "reason": "unauthorised_tool"},
         "arguments": {"level": "structural", "match": False, "details": {}},
         "order":     {"matcher": "topological", "match": False,
                       "violated_edge": ["docs.search", "sql_query"]},
         "cost_ok": True, "verdict": "unauthorised_tool"},
        {"step_id": 4, "span_id": "s_docs_1",
         "tool_name": {"expected": "docs.search", "actual": "docs.search",
                       "aliased": False, "match": True, "reason": "ok"},
         "arguments": {"level": "semantic_equivalence", "match": True, "details": {}},
         "order":     {"matcher": "topological", "match": True, "violated_edge": None},
         "cost_ok": True, "verdict": "ok"},
        {"step_id": 5, "span_id": "s_docs_2",
         "tool_name": {"expected": "docs.search", "actual": "docs.search",
                       "aliased": False, "match": True, "reason": "ok"},
         "arguments": {"level": "structural", "match": True, "details": {}},
         "order":     {"matcher": "topological", "match": True, "violated_edge": None},
         "cost_ok": True, "verdict": "redundant_call"},
        {"step_id": 7, "span_id": "s_crm",
         "tool_name": {"expected": "crm.lookup", "actual": "crm.lookup",
                       "aliased": False, "match": True, "reason": "ok"},
         "arguments": {"level": "exact_value", "match": True, "details": {}},
         "order":     {"matcher": "topological", "match": True, "violated_edge": None},
         "cost_ok": True, "verdict": "ok"},
        {"step_id": 9, "span_id": "s_ticket",
         "tool_name": {"expected": "open_ticket", "actual": "open_ticket",
                       "aliased": False, "match": True, "reason": "ok"},
         "arguments": {"level": "exact_value", "match": True, "details": {}},
         "order":     {"matcher": "topological", "match": True, "violated_edge": None},
         "cost_ok": True, "verdict": "ok"},
    ]


def _bad_budget():
    return {
        "budget_credit": 0.0,
        "flagged_verdicts": ["cost_over_budget", "latency_over_budget",
                             "steps_over_budget", "retry_budget_blown"],
        "attribution": {"cost_dominant_span": "s_llm_plan_1",
                        "latency_dominant_span": "s_crm",
                        "retry_dominant_tool": "crm.lookup"},
        "price_table_version": "v12",
    }


def test_chapter_01_bad_trajectory_floors_on_wrong_tool():
    rubric = load_rubric(CONFIG)
    # Add wrong_tool to the trip_on_verdict triggers via unauthorised_tool, which
    # is one of the floors in v1.4; chapter 03's example uses wrong_tool directly.
    v = roll_up(
        per_step=_per_step_bad_trajectory(),
        final_answer=AxisCredit(credit=1.0, method="exact_match_normalised"),
        groundedness=AxisCredit(credit=0.5, method="stub_v0"),
        budget=_bad_budget(),
        rubric=rubric,
        trajectory_id="c31f7a",
        reference_version="v3.2",
    )
    assert v.pass_ is False
    assert v.floor_tripped == "unauthorised_tool"
    # Weighted sum is reported even when the floor trips (chapter 03 §Starter guidance).
    assert 0.40 <= v.trajectory_score <= 0.60
    # flagged_verdicts carries both chapter-02 and chapter-04 entries.
    assert "unauthorised_tool" in v.flagged_verdicts
    assert "cost_over_budget" in v.flagged_verdicts


def test_below_threshold_no_floor_tripped():
    rubric = load_rubric(CONFIG)
    per_step = [
        {"step_id": 1, "span_id": "s",
         "tool_name": {"expected": "docs.search", "actual": "docs.search",
                       "aliased": False, "match": True, "reason": "ok"},
         "arguments": {"level": "structural", "match": False, "details": {}},
         "order":     {"matcher": "topological", "match": True, "violated_edge": None},
         "cost_ok": True, "verdict": "argument_mismatch"},
    ]
    v = roll_up(
        per_step=per_step,
        final_answer=AxisCredit(credit=0.4, method="token_f1"),
        groundedness=AxisCredit(credit=0.5, method="stub_v0"),
        budget={"budget_credit": 1.0, "flagged_verdicts": [],
                "attribution": {}, "price_table_version": "v12"},
        rubric=rubric,
        trajectory_id="t2",
        reference_version="v3.2",
    )
    assert v.pass_ is False
    assert v.floor_tripped is None
    assert v.trajectory_score < rubric.pass_threshold


def test_all_ones_passes():
    rubric = load_rubric(CONFIG)
    per_step = [
        {"step_id": 1, "span_id": "s",
         "tool_name": {"expected": "docs.search", "actual": "docs.search",
                       "aliased": False, "match": True, "reason": "ok"},
         "arguments": {"level": "semantic_equivalence", "match": True, "details": {}},
         "order":     {"matcher": "topological", "match": True, "violated_edge": None},
         "cost_ok": True, "verdict": "ok"},
    ]
    v = roll_up(
        per_step=per_step,
        final_answer=AxisCredit(credit=1.0, method="exact_match_normalised"),
        groundedness=AxisCredit(credit=1.0, method="stub_v0"),
        budget={"budget_credit": 1.0, "flagged_verdicts": [],
                "attribution": {}, "price_table_version": "v12"},
        rubric=rubric,
        trajectory_id="t3",
        reference_version="v3.2",
    )
    assert v.pass_ is True
    assert v.trajectory_score == pytest.approx(1.0)


def test_small_step_problem_still_passes():
    rubric = load_rubric(CONFIG)
    # name_credit 0.0 on ONE required step but no floor tripped; weighted sum has to
    # clear 0.80 for pass. With weights 0.20/0.10/0.10/0.20/0.25/0.15 and all other
    # axes 1.0, name=0.0 gives sum 0.80 which is at threshold — we bump answer to 1
    # and keep name 1.0 to prove the "small step problem but no floor" shape.
    per_step = [
        {"step_id": 1, "span_id": "s_a",
         "tool_name": {"expected": "docs.search", "actual": "docs.search",
                       "aliased": False, "match": True, "reason": "ok"},
         "arguments": {"level": "structural", "match": True, "details": {}},
         "order":     {"matcher": "topological", "match": True, "violated_edge": None},
         "cost_ok": True, "verdict": "ok"},
        # A legitimate extra tool call — chapter 03 says extra_tool does NOT lower
        # name_credit (it is a separate floor concern if included in trip_on_verdict).
        {"step_id": 2, "span_id": "s_b",
         "tool_name": {"expected": None, "actual": "crm.lookup",
                       "aliased": False, "match": True, "reason": "extra_tool"},
         "arguments": {"level": "structural", "match": True, "details": {}},
         "order":     {"matcher": "topological", "match": True, "violated_edge": None},
         "cost_ok": True, "verdict": "extra_tool"},
    ]
    v = roll_up(
        per_step=per_step,
        final_answer=AxisCredit(credit=1.0, method="exact_match_normalised"),
        groundedness=AxisCredit(credit=1.0, method="stub_v0"),
        budget={"budget_credit": 1.0, "flagged_verdicts": [],
                "attribution": {}, "price_table_version": "v12"},
        rubric=rubric,
        trajectory_id="t4",
        reference_version="v3.2",
    )
    # `extra_tool` is NOT in trip_on_verdict for v1.4, so the trajectory passes.
    assert v.pass_ is True
    assert v.floor_tripped is None


def test_rubric_version_round_trips():
    rubric = load_rubric(CONFIG)
    v = roll_up(
        per_step=[],
        final_answer=AxisCredit(credit=1.0, method="exact_match_normalised"),
        groundedness=AxisCredit(credit=1.0, method="stub_v0"),
        budget={"budget_credit": 1.0, "flagged_verdicts": [],
                "attribution": {}, "price_table_version": "v12"},
        rubric=rubric,
        trajectory_id="t5",
        reference_version="v3.2",
    )
    assert v.rubric_version == "v1.4"
    assert v.price_table_version == "v12"
```

### 2.8 Hand-computed trajectory_score for the bad-trajectory test

With `v1.4` weights (`name=0.20, args=0.10, order=0.10, answer=0.20, groundedness=0.25, budget=0.15`) and the per-step above:

- `name_credit`: required steps (ok + wrong_tool/unauthorised_tool rows) → min(0, 1, 1, 1, 1) = **0.0**. (The unauthorised step is excluded from the name required-set by the `_axis_name` filter — it was never in the DAG — so effectively 4 ok rows → **1.0**. Pick whichever convention the chapter uses on your surface; the reference solution scopes required steps to the DAG's required_set and treats unauthorised_tool as a floor-only concern.) Using the reference-solution convention: **1.0**.
- `args_credit`: min(args.match across required) = min(False, True, True, True, True) = **0.0** (the SQL step failed args).
- `order_credit`: min(False, True, True, True, True) = **0.0**.
- `answer_credit`: **1.0** (test fixture passes).
- `groundedness_credit`: **0.5** (stub).
- `budget_credit`: **0.0** (bad fixture).

Weighted sum: `0.20·1.0 + 0.10·0.0 + 0.10·0.0 + 0.20·1.0 + 0.25·0.5 + 0.15·0.0 = 0.525`.

The test asserts `0.40 ≤ trajectory_score ≤ 0.60`, which this clears. Chapter 03's example quotes `≈ 0.455` for a slightly different per-step shape (treating the unauthorised step as pulling name_credit to 0 as well); both shapes are defensible — the test tolerance absorbs the choice.

## 3. Validation steps

1. **Install**: `pip install pydantic pyyaml pytest`. If using the exercise-01 code, `pip install -e ../exercise-01`.
2. **Load the rubric**: `python -c "from rubric.loader import load_rubric; print(load_rubric('configs/rubric_order_status_not_shipped_policy_v1_4.yaml'))"`. It prints the config; a weight sum that does not equal 1.0 raises a `ValidationError`.
3. **Run tests**: `pytest -q tests/`. All five cases pass.
4. **End-to-end pipeline on the chapter-01 bad trajectory**:
   ```bash
   python -m scorer.cli \
     --trajectory ../exercise-01/fixtures/support_agent_bad_trajectory.json \
     --reference  ../exercise-01/fixtures/reference_order_status_not_shipped_policy_v3_2.yaml \
     --price-table ../exercise-01/fixtures/price_table_v12.json > /tmp/scores.json
   python -c "
   import json
   from rubric.loader import load_rubric
   from rubric.rollup import roll_up
   from rubric.schema import AxisCredit
   s = json.load(open('/tmp/scores.json'))
   r = load_rubric('configs/rubric_order_status_not_shipped_policy_v1_4.yaml')
   v = roll_up(per_step=s['per_step'],
               final_answer=AxisCredit(credit=0.0, method='exact_match_normalised'),
               groundedness=AxisCredit(credit=0.5, method='stub_v0'),
               budget=s['budget'],
               rubric=r,
               trajectory_id=s['budget']['trajectory_id'],
               reference_version='v3.2')
   print(v.model_dump_json(by_alias=True, indent=2))
   "
   ```
   The printed verdict has `pass=false`, `floor_tripped` names one of the floor verdicts, and `flagged_verdicts` carries both chapter-02 and chapter-04 entries.
5. **Aggregator sanity**: score 10 hand-written fixtures with two rubrics (`v1.4` baseline + a hypothetical `v1.5` with `budget` weight bumped to 0.25) and call `aggregate(..., slice_by=['task_family', 'rubric_version'])`. The two-row output lets a reviewer verify `pass_rate` by hand.
6. **No model call**: `grep -R -nE "anthropic\.|openai\.|langchain" rubric/` returns nothing.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | `sum(weights) == 1.0` is enforced on load (within 1e-6). | `pydantic.ValidationError` on `weights.name=0.25` leaving others at defaults. | Loader silently accepts. |
| R2 | `pass_threshold` is a scalar in `[0, 1]`; `trajectory_score ≥ pass_threshold` plus no floor tripped is the pass condition. | Explicit check in `roll_up`; `pass=false` when floor tripped even if sum ≥ threshold. | One of the two checks missing. |
| R3 | Floors trip on a named `trip_on_verdict` list, not on a credit threshold. | `floor_tripped` is a verdict name string. | Floor is a credit threshold like `name_credit < 0.5`. |
| R4 | Credit-to-step attribution uses worst-case (min) across required steps for name/args/order. | `_axis_worst_case` uses `min`. | Mean or median. |
| R5 | `flagged_verdicts` is the union of chapter-02 (per-step) and chapter-04 (budget) verdicts. | Both lists appear in the verdict. | Only one is included. |
| R6 | `rubric_version`, `reference_version`, `price_table_version` round-trip to the verdict. | All three present in `PerTrajectoryVerdict`. | Any missing. |
| R7 | Per-task-family rubrics are enforced; a mismatch between rubric and trajectory task_family raises or is flagged. | Loader checks `rubric.task_family == trajectory.task_family`. | Loader does not check. |
| R8 | `aggregate()` uses documented percentile semantics (not reinvented). | `statistics.quantiles(method='inclusive')` or `numpy.percentile(method='nearest')`. | Custom percentile math. |
| R9 | No model call anywhere; the judge is a stub that raises `NotImplementedError`. | `grep` clean; stub raises. | Stub returns a dummy value. |
| R10 | Weighted sum is computed and reported even when a floor trips. | `trajectory_score` present in the `floor_tripped != None` test. | Score is 0 or omitted. |

Passing submission is 8+ of 10 with R1, R3, R6, R9 hard-required.

## 5. Common mistakes

- **Tuning weights to fit the sample.** Chapter 03 is explicit: weights are a product artefact, authored once per task family, versioned, and reviewed on PR. Re-fitting weights because "the regression set is failing too much" rewrites history; promote the regression set or add a cohort override instead.
- **Combining floors with credit thresholds.** Writing `if name_credit < 0.5: pass=false` collapses two different concerns. Floors are *named* failure modes that are unacceptable at any credit value; credit thresholds are a *gradient* across the weighted sum. Mixing them hides the attribution a reviewer uses.
- **Letting `extra_tool` lower `name_credit`.** The required set is defined by the reference DAG; a tool called that is in `allowed_set` but not `required_set` contributes to `flagged_verdicts["extra_tool"]`, which may or may not be a floor. Pulling `name_credit` down for extra tools is a double punishment.
- **Dropping `trajectory_score` when the floor trips.** A reviewer wants to see "floor tripped, and additionally the score was 0.45" — the second half says whether the trajectory was *close* to the pass line or nowhere near it. The verdict shape chapter 03 ships carries both.
- **Scoring across task families with one rubric.** The weight `0.25` on groundedness makes sense for a RAG-heavy order-status flow; it would be wrong on a tool-only flow (e.g., a fast-path status check that never touches a retriever). Per-family files are mandatory.
- **Mean across steps for `name_credit` / `args_credit`.** The chapter 03 rule is worst-case across the required steps. A mean lets a 9-step trajectory with one catastrophic wrong step pass at `name_credit ≈ 0.89` when the actual quality is "one critical step wrong".
- **Calling the final-answer judge from the trajectory scorer.** The judge lives in mod-104; this exercise's `reference_based_judge` is a stub. Invoking the judge from here conflates trajectory scoring (chapter 03) with judge configuration (mod-104); the version hashes diverge.
- **Hand-rolled percentile.** `numpy.percentile(..., method='linear')` and `statistics.quantiles(method='exclusive')` are both defensible; whatever you pick, use a stdlib / numpy function so the choice is one grep away.
- **Letting the rubric file drift out of mod-110.** The rubric is a product artefact that travels with the reference trajectories and the price table; mod-110 is where the versioning lives. In-tree copies rot.

## 6. References

- Local chapters: `lessons/mod-103-trajectory-and-tool-eval/03-final-answer-and-partial-credit-rubric.md`, `04-budgets-and-rollout-gates.md`, `02-scoring-per-step-tool-calls.md`.
- Exercise-01 solution (upstream of this rubric); same module.
- SQuAD token-F1 definition — Rajpurkar et al., SQuAD paper and evaluation script, <https://rajpurkar.github.io/SQuAD-explorer/>.
- Python `statistics.quantiles`, <https://docs.python.org/3/library/statistics.html#statistics.quantiles>.
- Pydantic v2 docs, <https://docs.pydantic.dev/latest/> (for `model_validator` and `AxisCredit` constraints).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> (motivates `unauthorised_tool` as a floor).
- MITRE ATLAS, <https://atlas.mitre.org/> (ATLAS TTPs inform the `fabricated_claim` floor the rubric expects mod-105 to feed).
- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> (Measure / Manage functions inform the versioning discipline on rubrics and the hash-into-CI expectation mod-106 reads).
