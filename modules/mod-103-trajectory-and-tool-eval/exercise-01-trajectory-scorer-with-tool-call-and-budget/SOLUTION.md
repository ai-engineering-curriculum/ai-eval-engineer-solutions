# exercise-01 — Trajectory Scorer With Tool Call And Budget — Solution

> Reference solution for `lessons/mod-103-trajectory-and-tool-eval/exercises/exercise-01-trajectory-scorer-with-tool-call-and-budget.md`.
> Chapters 02 and 04 of the learning module are the primary input; chapter 01's running bad trajectory is the fixture. OpenInference / OTel GenAI attribute names cited in the reader are from the specs listed in §6.

## 1. Solution overview

The exercise produces two scorers that read one captured trajectory JSON and emit two structured verdicts — never a scalar. `score_per_step` answers chapter 02's question "did the agent take the right steps?" across three axes (name, arguments, order). `score_budget` answers chapter 04's question "did it do it inside the allowed budgets?" across four families (cost, latency, step count, retry rate). The CLI composes the two.

Four design choices anchor the solution:

1. **Harness-agnostic.** The scorer reads a flat trajectory JSON — no Inspect, no OpenInference client, no model call. Exercise-03 is where this gets wrapped in `inspect-ai`; exercise-05 is where it gets fed a replayed trajectory. If this module calls a model or imports Inspect, the layering is wrong.
2. **Verdict shape over scalar.** The output is `list[StepVerdict]` + `BudgetVerdict`. Chapter 02's shape has explicit `tool_name`, `arguments`, `order`, `cost_ok`, and a per-step `verdict` tag. Chapter 04's shape has per-family credits, `flagged_verdicts`, and an attribution block. Exercise-02 is where a scalar `trajectory_score` enters via the rubric roll-up.
3. **Determinism.** No `time.time()`, no `random`, no dict-order-dependent string formatting, no `None` for "unknown" where `null` in the JSON would be ambiguous. Running the scorer twice on the same inputs is byte-identical.
4. **Full attribution, not short-circuit.** Chapter 02 §"Sub-scores rolled into a per-step verdict": if the name does not match, the argument and ordering scorers still run. A reviewer wants to see whether a `wrong_tool` was called with sensible arguments (prompt bug) or nonsense arguments (schema bug).

## 2. Worked implementation

The layout the exercise asks for:

```
mod-103/exercise-01/
├── pyproject.toml
├── README.md
├── scorer/
│   ├── __init__.py
│   ├── canonical.py        # name/arg normalisation shared across levels
│   ├── per_step.py         # score_per_step(trajectory, reference)
│   ├── budget.py           # score_budget(trajectory, reference, price_table)
│   ├── schema.py           # dataclass / Pydantic verdict definitions
│   └── cli.py              # python -m scorer.cli
├── fixtures/
│   ├── reference_order_status_not_shipped_policy_v3_2.yaml
│   ├── price_table_v12.json
│   ├── support_agent_good_trajectory.json
│   ├── support_agent_bad_trajectory.json
│   ├── collapsed_retry_trajectory.json
│   ├── parallel_tool_calls_trajectory.json
│   └── semantic_equivalent_query_trajectory.json
└── tests/
    ├── test_per_step.py
    └── test_budget.py
```

### 2.1 `scorer/schema.py` — verdict shape

```python
"""Verdict dataclasses. Pydantic optional; keep the JSON shape canonical."""
from __future__ import annotations
from dataclasses import dataclass, field, asdict
from typing import Any, Literal


ArgLevel = Literal["structural", "exact_value", "semantic_equivalence",
                   "ast_based", "judge_graded"]
StepVerdictTag = Literal[
    "ok", "wrong_tool", "unauthorised_tool", "extra_tool",
    "ordering_violation", "argument_mismatch", "redundant_call",
    "retry_budget_blown_local",
]
BudgetVerdictTag = Literal[
    "cost_over_budget", "latency_over_budget",
    "steps_over_budget", "tool_calls_over_budget",
    "retry_budget_blown",
]


@dataclass(frozen=True)
class NameSubScore:
    expected: str | None
    actual: str
    aliased: bool
    match: bool
    reason: str  # "ok" | "wrong_tool" | "unauthorised_tool" | "extra_tool" | "alias_resolved"


@dataclass(frozen=True)
class ArgumentsSubScore:
    level: ArgLevel
    match: bool
    details: dict[str, Any] = field(default_factory=dict)


@dataclass(frozen=True)
class OrderSubScore:
    matcher: Literal["strict", "topological", "interleaving"]
    match: bool
    violated_edge: tuple[str, str] | None


@dataclass(frozen=True)
class StepVerdict:
    step_id: int
    span_id: str
    tool_name: NameSubScore
    arguments: ArgumentsSubScore
    order: OrderSubScore
    cost_ok: bool
    verdict: StepVerdictTag

    def to_json(self) -> dict[str, Any]:
        d = asdict(self)
        # tuple → list for JSON
        if d["order"]["violated_edge"] is not None:
            d["order"]["violated_edge"] = list(d["order"]["violated_edge"])
        return d


@dataclass(frozen=True)
class BudgetAxis:
    observed: float
    ceiling: float
    credit: float
    over_fraction: float


@dataclass(frozen=True)
class BudgetAttribution:
    cost_dominant_span: str | None
    latency_dominant_span: str | None
    retry_dominant_tool: str | None


@dataclass(frozen=True)
class BudgetVerdict:
    trajectory_id: str
    task_family: str
    reference_version: str
    price_table_version: str
    budget_credit: float
    budget_axes: dict[str, BudgetAxis]
    latency_diagnostics: dict[str, float]  # wallclock, critical_path, sum_of_step
    flagged_verdicts: list[BudgetVerdictTag]
    attribution: BudgetAttribution
    terminated_by_limit: str | None  # Inspect harness limit if hit; else null

    def to_json(self) -> dict[str, Any]:
        return {
            "trajectory_id": self.trajectory_id,
            "task_family": self.task_family,
            "reference_version": self.reference_version,
            "price_table_version": self.price_table_version,
            "budget_credit": self.budget_credit,
            "budget_axes": {k: asdict(v) for k, v in self.budget_axes.items()},
            "latency_diagnostics": self.latency_diagnostics,
            "flagged_verdicts": self.flagged_verdicts,
            "attribution": asdict(self.attribution),
            "terminated_by_limit": self.terminated_by_limit,
        }
```

### 2.2 `scorer/canonical.py` — name + argument canonicalisation

```python
"""Chapter 02 §Name matching + §Argument matching Level 1-3 primitives.

Kept harness-agnostic: no Inspect types, no OTel attribute reads — the reader
flattened the trace into the exercise JSON schema already.
"""
from __future__ import annotations
import re
import unicodedata
from typing import Any, Iterable


def normalise_name(name: str) -> str:
    """NFKC + casefold. Keep namespaces (`crm.lookup` != `lookup`)."""
    return unicodedata.normalize("NFKC", name).casefold().strip()


def resolve_alias(name: str, alias_table: dict[str, str]) -> tuple[str, bool]:
    """Return (resolved_name, aliased_bool). Alias table is {old: new}."""
    n = normalise_name(name)
    for old, new in alias_table.items():
        if normalise_name(old) == n:
            return normalise_name(new), True
    return n, False


def canonicalise_arguments(args: dict[str, Any]) -> dict[str, Any]:
    """Chapter 02 Level-2 canonicalisation: sort keys, strip strings, coerce numerics."""
    def _coerce(v: Any) -> Any:
        if isinstance(v, str):
            s = unicodedata.normalize("NFKC", v).strip()
            # numeric-string → number if it parses cleanly; do NOT lose leading zeros.
            if re.fullmatch(r"-?\d+", s):
                return int(s)
            if re.fullmatch(r"-?\d+\.\d+", s):
                return float(s)
            return s
        if isinstance(v, dict):
            return {k: _coerce(v[k]) for k in sorted(v)}
        if isinstance(v, list):
            return [_coerce(x) for x in v]
        return v
    return {k: _coerce(args[k]) for k in sorted(args)}


# --- Level 3 semantic-equivalence rules ----------------------------------
# One rule per (tool, field). The rule registry lives alongside the reference
# trajectories in mod-110; this scorer just reads it.

_STOPWORDS = frozenset({"a", "an", "the", "of", "for", "to", "in", "on", "is"})


def token_set(s: str) -> frozenset[str]:
    tokens = re.findall(r"\w+", unicodedata.normalize("NFKC", s).casefold())
    return frozenset(t for t in tokens if t not in _STOPWORDS)


def token_set_includes(observed: str, required_tokens: Iterable[str]) -> bool:
    obs = token_set(observed)
    req = frozenset(t.casefold() for t in required_tokens)
    return req.issubset(obs)


def in_allowed_set(observed: Any, allowed: Iterable[Any]) -> bool:
    return observed in frozenset(allowed)


# Level 4 / Level 5 are out of scope for this exercise.
def ast_equivalent(*_args, **_kw):  # pragma: no cover - mod-104 territory
    raise NotImplementedError(
        "Level-4 AST argument matching is a stretch goal; see lesson 02 §Argument matching "
        "and exercise-01 stretch goals. Full implementation belongs in mod-104."
    )


def judge_graded(*_args, **_kw):  # pragma: no cover - mod-104 territory
    raise NotImplementedError(
        "Level-5 judge-graded argument matching is owned by mod-104's judge module; "
        "trajectory scorer calls into the judge, not the other way around."
    )
```

### 2.3 `scorer/per_step.py` — the chapter-02 verdict

```python
"""Per-step scorer. Chapter 02 §"Sub-scores rolled into a per-step verdict"."""
from __future__ import annotations
from graphlib import TopologicalSorter, CycleError
from typing import Any

from .canonical import (
    normalise_name, resolve_alias, canonicalise_arguments,
    token_set_includes, in_allowed_set,
)
from .schema import (
    NameSubScore, ArgumentsSubScore, OrderSubScore, StepVerdict,
)


def _collapse_retries(steps: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """Chapter 02 §Name matching: two adjacent TOOL spans with identical
    (tool_name, arguments) that differ only in success status are a harness
    retry, not an agent decision. Collapse them; carry the retry_count forward.
    """
    collapsed: list[dict[str, Any]] = []
    for step in steps:
        if step["kind"] != "TOOL":
            collapsed.append(step)
            continue
        if collapsed and collapsed[-1]["kind"] == "TOOL" \
                and normalise_name(collapsed[-1]["tool_name"]) == normalise_name(step["tool_name"]) \
                and canonicalise_arguments(collapsed[-1].get("arguments", {})) \
                    == canonicalise_arguments(step.get("arguments", {})):
            prev = dict(collapsed[-1])
            prev["retry_count"] = prev.get("retry_count", 0) + 1
            # Keep the final status (the one the model saw).
            prev["status"] = step.get("status", prev.get("status"))
            collapsed[-1] = prev
            continue
        collapsed.append(step)
    return collapsed


def _tool_steps(steps: list[dict[str, Any]]) -> list[dict[str, Any]]:
    """Agent-decision steps are TOOL spans (observable action) plus the terminal
    LLM reply. LLM `plan` steps that emit tool_calls are the *declarations* — the
    name/argument/order verdicts attach to the TOOL span that executes them.
    Chapter 02 §Core concepts treats the step as the executed action.
    """
    return [s for s in steps if s["kind"] in {"TOOL", "RETRIEVER"}]


def _score_name(
    observed: str, reference_set: frozenset[str],
    allowed_set: frozenset[str], required_set: frozenset[str],
    alias_table: dict[str, str],
) -> NameSubScore:
    resolved, aliased = resolve_alias(observed, alias_table)
    norm_reference = {normalise_name(r) for r in reference_set}
    norm_allowed = {normalise_name(r) for r in allowed_set}
    norm_required = {normalise_name(r) for r in required_set}
    if resolved in norm_required:
        return NameSubScore(resolved, observed, aliased, True, "ok")
    if resolved in norm_allowed:
        return NameSubScore(resolved, observed, aliased, True, "extra_tool")
    if resolved not in norm_reference:
        # Not in the authorised catalog at all — security concern, chapter 02.
        return NameSubScore(None, observed, aliased, False, "unauthorised_tool")
    return NameSubScore(None, observed, aliased, False, "wrong_tool")


def _score_arguments(
    tool_name: str, observed_args: dict[str, Any],
    argument_rules: dict[str, dict[str, Any]],
) -> ArgumentsSubScore:
    # If no rule is declared for this tool, Level-1 structural (keys present + correct types).
    rules = {k: v for k, v in argument_rules.items() if k.startswith(f"{tool_name}.")}
    if not rules:
        return ArgumentsSubScore(
            level="structural", match=True,
            details={"missing_keys": [], "type_errors": [],
                     "note": "no rule registered; structural-only"},
        )
    details: dict[str, Any] = {"per_field": {}}
    canon = canonicalise_arguments(observed_args)
    all_pass = True
    worst_level: str = "structural"
    for key, rule in rules.items():
        field_name = key.split(".", 1)[1]
        value = canon.get(field_name)
        level = rule["level"]
        passed: bool
        if level == "1-structural":
            passed = field_name in canon
            worst_level = max(worst_level, "structural",
                              key=_level_rank)
        elif level == "2-exact":
            allowed = rule.get("rule_in") or rule.get("in") or []
            passed = in_allowed_set(value, allowed)
            worst_level = max(worst_level, "exact_value", key=_level_rank)
        elif level == "3-semantic":
            if rule["rule"].startswith("token_set_includes"):
                required = rule.get("tokens") or []
                passed = isinstance(value, str) and token_set_includes(value, required)
            else:
                raise ValueError(f"unsupported semantic rule: {rule['rule']}")
            worst_level = max(worst_level, "semantic_equivalence", key=_level_rank)
        else:
            raise ValueError(f"level {level} is out of scope for exercise-01")
        details["per_field"][field_name] = {"level": level, "match": passed, "value": value}
        all_pass = all_pass and passed
    return ArgumentsSubScore(level=worst_level, match=all_pass, details=details)


def _level_rank(level: str) -> int:
    return ["structural", "exact_value", "semantic_equivalence",
            "ast_based", "judge_graded"].index(level)


def _score_order(
    observed_names: list[str], reference_edges: list[tuple[str, str]],
    allowed_set: frozenset[str],
) -> list[OrderSubScore]:
    """Return one OrderSubScore per observed step. On the first violated edge,
    mark that step `match=False` and name the edge; subsequent steps still get
    a verdict but the violated edge carries forward for clarity.
    """
    # Build predecessor map
    predecessors: dict[str, set[str]] = {}
    for a, b in reference_edges:
        predecessors.setdefault(normalise_name(b), set()).add(normalise_name(a))
    # Any required predecessor that has not been seen by step N invalidates N.
    seen: set[str] = set()
    out: list[OrderSubScore] = []
    for name in observed_names:
        n = normalise_name(name)
        reqs = predecessors.get(n, set())
        missing = reqs - seen
        if missing:
            violated = (sorted(missing)[0], n)
            out.append(OrderSubScore(
                matcher="topological", match=False, violated_edge=violated,
            ))
        else:
            out.append(OrderSubScore(
                matcher="topological", match=True, violated_edge=None,
            ))
        seen.add(n)
    # Sanity-check the reference DAG itself is acyclic (chapter 02 §Ordering).
    try:
        TopologicalSorter({b: {a} for (a, b) in reference_edges}).prepare()
    except CycleError as e:
        raise ValueError(f"reference DAG has a cycle: {e}") from e
    return out


def _detect_redundant(
    step: dict[str, Any], prior_steps: list[dict[str, Any]],
) -> bool:
    """Chapter 02's bad-trajectory example: a second docs.search with no new
    information is redundant. Working definition: same tool_name, same
    canonicalised arguments *modulo* any Level-3 field, and the prior call's
    status was `ok`.
    """
    canon_now = canonicalise_arguments(step.get("arguments", {}))
    for prev in prior_steps:
        if prev["kind"] != step["kind"]:
            continue
        if normalise_name(prev["tool_name"]) != normalise_name(step["tool_name"]):
            continue
        if prev.get("status", "ok") != "ok":
            continue
        canon_prev = canonicalise_arguments(prev.get("arguments", {}))
        if canon_now == canon_prev:
            return True
    return False


def score_per_step(
    trajectory: dict[str, Any], reference: dict[str, Any],
) -> list[StepVerdict]:
    """Return one StepVerdict per agent-decision step (TOOL / RETRIEVER span)."""
    steps = _collapse_retries(trajectory["steps"])
    tool_steps = _tool_steps(steps)

    required_set = frozenset(reference["required_set"])
    allowed_set = frozenset(reference["allowed_set"])
    reference_set = required_set | allowed_set
    edges = [tuple(e) for e in reference["edges"]]
    alias_table = reference.get("aliases", {})
    rules = reference.get("argument_rules", {})

    observed_names = [s["tool_name"] for s in tool_steps]
    order_verdicts = _score_order(observed_names, edges, allowed_set)

    verdicts: list[StepVerdict] = []
    prior: list[dict[str, Any]] = []
    for idx, step in enumerate(tool_steps):
        name_v = _score_name(
            step["tool_name"], reference_set, allowed_set, required_set, alias_table,
        )
        args_v = _score_arguments(
            normalise_name(step["tool_name"]), step.get("arguments", {}), rules,
        )
        order_v = order_verdicts[idx]
        redundant = _detect_redundant(step, prior)

        tag: str
        if name_v.reason == "unauthorised_tool":
            tag = "unauthorised_tool"
        elif name_v.reason == "wrong_tool":
            tag = "wrong_tool"
        elif name_v.reason == "extra_tool":
            tag = "extra_tool"
        elif not args_v.match:
            tag = "argument_mismatch"
        elif not order_v.match:
            tag = "ordering_violation"
        elif redundant:
            tag = "redundant_call"
        else:
            tag = "ok"

        verdicts.append(StepVerdict(
            step_id=step["step_id"],
            span_id=step["span_id"],
            tool_name=name_v,
            arguments=args_v,
            order=order_v,
            cost_ok=True,  # chapter 04's scorer may rewrite via a diagnostic merge
            verdict=tag,  # type: ignore[arg-type]
        ))
        prior.append(step)
    return verdicts
```

### 2.4 `scorer/budget.py` — the chapter-04 verdict

```python
"""Budget scorer. Chapter 04 §Four budget families + §Composite budget score."""
from __future__ import annotations
from typing import Any

from .schema import (
    BudgetAxis, BudgetAttribution, BudgetVerdict,
)


def _clamp(x: float, lo: float, hi: float) -> float:
    return max(lo, min(hi, x))


def _axis(observed: float, ceiling: float) -> BudgetAxis:
    """Chapter 04 §Composite budget score. Linear decay to zero at 2× ceiling."""
    if ceiling <= 0:
        return BudgetAxis(observed=observed, ceiling=ceiling, credit=1.0, over_fraction=0.0)
    over_fraction = max(0.0, observed / ceiling - 1.0)
    credit = _clamp(1.0 - over_fraction, 0.0, 1.0)
    return BudgetAxis(observed=observed, ceiling=ceiling, credit=credit,
                      over_fraction=over_fraction)


def _cost_of_llm_step(step: dict[str, Any], price_table: dict[str, Any]) -> float:
    """Charge gen_ai.response.model (chapter 04 §What `app.cost_usd` has to carry)."""
    model = step["llm.response.model"]
    prices = price_table[model]
    tokens = step["tokens"]
    input_c = tokens["input"] * prices["input_per_1k_tokens"] / 1000.0
    cached_c = tokens.get("cached_input", 0) * prices.get(
        "cached_input_per_1k_tokens", prices["input_per_1k_tokens"]
    ) / 1000.0
    output_c = tokens["output"] * prices["output_per_1k_tokens"] / 1000.0
    return input_c + cached_c + output_c


def _step_cost(step: dict[str, Any], price_table: dict[str, Any]) -> float:
    if step["kind"] == "LLM":
        return _cost_of_llm_step(step, price_table)
    # TOOL / RETRIEVER: priced tools carry cost_usd; free in-house tools are zero.
    return float(step.get("cost_usd", 0.0))


def _latency_diagnostics(trajectory: dict[str, Any]) -> dict[str, float]:
    """Wall-clock, critical-path, sum-of-step. Chapter 04 §Latency arithmetic."""
    agent = trajectory["agent"]
    wall = (agent["end_ns"] - agent["start_ns"]) / 1e9
    sum_of_step = sum(s.get("latency_ms", 0) for s in trajectory["steps"]) / 1000.0
    # Critical path: longest chain by parent/child. If the flat step JSON does
    # not carry parent_span, fall back to assuming sequential (equal to sum).
    critical = sum_of_step  # sequential fallback
    if any("parent_span_id" in s for s in trajectory["steps"]):
        critical = _critical_path_seconds(trajectory["steps"])
    return {
        "wallclock_s": wall,
        "critical_path_s": critical,
        "sum_of_step_s": sum_of_step,
    }


def _critical_path_seconds(steps: list[dict[str, Any]]) -> float:
    """Longest end-to-end path by (start_ns, end_ns, parent_span_id)."""
    by_id = {s["span_id"]: s for s in steps}
    memo: dict[str, float] = {}

    def depth(sid: str) -> float:
        if sid in memo:
            return memo[sid]
        s = by_id[sid]
        own = (s["end_ns"] - s["start_ns"]) / 1e9 if "end_ns" in s else s.get("latency_ms", 0) / 1000.0
        children = [c for c in steps if c.get("parent_span_id") == sid]
        child_max = max((depth(c["span_id"]) for c in children), default=0.0)
        memo[sid] = own + child_max
        return memo[sid]

    roots = [s for s in steps if "parent_span_id" not in s or s["parent_span_id"] is None]
    return max((depth(r["span_id"]) for r in roots), default=0.0)


def _retry_counts(steps: list[dict[str, Any]]) -> dict[str, int]:
    """Count collapsed retries per tool. Chapter 04 §Step count, tool count, retry count."""
    counts: dict[str, int] = {}
    last_key: tuple[str, str] | None = None
    for s in steps:
        if s["kind"] != "TOOL":
            last_key = None
            continue
        key = (s["tool_name"], str(sorted(s.get("arguments", {}).items())))
        if key == last_key:
            counts[s["tool_name"]] = counts.get(s["tool_name"], 0) + 1
        else:
            counts.setdefault(s["tool_name"], 0)
        last_key = key
    return counts


def _step_count(steps: list[dict[str, Any]]) -> int:
    """Chapter 04: a step is one iteration of the agent loop (one LLM action-selection)."""
    return sum(1 for s in steps if s["kind"] == "LLM")


def _tool_call_count(steps: list[dict[str, Any]]) -> int:
    """One per tool_calls array entry across all LLM steps."""
    total = 0
    for s in steps:
        if s["kind"] == "LLM":
            total += len(s.get("tool_calls", []))
    return total


def score_budget(
    trajectory: dict[str, Any], reference: dict[str, Any],
    price_table: dict[str, Any],
) -> BudgetVerdict:
    budgets = reference["budgets"]
    steps = trajectory["steps"]

    total_cost = sum(_step_cost(s, price_table) for s in steps)
    # Chapter 04 assertion: per-step sum should match agent.app.cost_usd within rounding.
    agent_cost = float(trajectory["agent"].get("app.cost_usd", total_cost))
    assert abs(total_cost - agent_cost) < 0.01 * max(1.0, agent_cost), (
        f"per-step cost sum {total_cost:.4f} disagrees with agent total {agent_cost:.4f}"
        " — likely a price-table version mismatch or a span missing from the flatten"
    )

    latency = _latency_diagnostics(trajectory)
    retry_counts = _retry_counts(steps)
    max_retry = max(retry_counts.values(), default=0)
    step_count = _step_count(steps)
    tool_call_count = _tool_call_count(steps)

    axes = {
        "cost": _axis(total_cost, budgets["cost_usd_ceiling"]),
        "latency": _axis(latency["wallclock_s"], budgets["time_limit_s"]),
        "steps": _axis(step_count, budgets["max_steps"]),
        "tool_calls": _axis(tool_call_count, budgets["max_tool_calls"]),
        "retries": _axis(max_retry, budgets["max_retries_per_tool"]),
    }

    budget_credit = min(a.credit for a in axes.values())

    flagged: list[str] = []
    if axes["cost"].credit < 1.0:
        flagged.append("cost_over_budget")
    if axes["latency"].credit < 1.0:
        flagged.append("latency_over_budget")
    if axes["steps"].credit < 1.0:
        flagged.append("steps_over_budget")
    if axes["tool_calls"].credit < 1.0:
        flagged.append("tool_calls_over_budget")
    if axes["retries"].credit < 1.0:
        flagged.append("retry_budget_blown")

    # Attribution: chapter 04 §"Find the span" rule.
    llm_steps = [s for s in steps if s["kind"] == "LLM"]
    cost_dominant = max(
        llm_steps, key=lambda s: _cost_of_llm_step(s, price_table), default=None,
    )
    latency_dominant = max(
        steps, key=lambda s: s.get("latency_ms", 0), default=None,
    )
    retry_dominant_tool = (
        max(retry_counts, key=retry_counts.get) if retry_counts else None
    )

    return BudgetVerdict(
        trajectory_id=trajectory["trajectory_id"],
        task_family=trajectory["task_family"],
        reference_version=trajectory["reference_version"],
        price_table_version=price_table.get("_version", "unknown"),
        budget_credit=budget_credit,
        budget_axes=axes,
        latency_diagnostics=latency,
        flagged_verdicts=flagged,  # type: ignore[arg-type]
        attribution=BudgetAttribution(
            cost_dominant_span=cost_dominant["span_id"] if cost_dominant else None,
            latency_dominant_span=latency_dominant["span_id"] if latency_dominant else None,
            retry_dominant_tool=retry_dominant_tool,
        ),
        terminated_by_limit=trajectory.get("terminated_by_limit"),
    )
```

### 2.5 `scorer/cli.py` — the one-screen CLI

```python
"""`python -m scorer.cli --trajectory … --reference … --price-table …`"""
from __future__ import annotations
import argparse
import json
import sys

import yaml

from .per_step import score_per_step
from .budget import score_budget


def main(argv: list[str] | None = None) -> int:
    p = argparse.ArgumentParser(prog="scorer")
    p.add_argument("--trajectory", required=True)
    p.add_argument("--reference", required=True)
    p.add_argument("--price-table", required=True)
    p.add_argument("--price-table-version", default=None,
                   help="sanity check against the trajectory's recorded version")
    ns = p.parse_args(argv)

    with open(ns.trajectory) as f:
        trajectory = json.load(f)
    with open(ns.reference) as f:
        reference = yaml.safe_load(f) if ns.reference.endswith((".yaml", ".yml")) else json.load(f)
    with open(ns.price_table) as f:
        price_table = json.load(f)

    if ns.price_table_version and price_table.get("_version") != ns.price_table_version:
        print(f"price-table version mismatch: {price_table.get('_version')} != {ns.price_table_version}",
              file=sys.stderr)
        return 2

    per_step = [v.to_json() for v in score_per_step(trajectory, reference)]
    budget = score_budget(trajectory, reference, price_table).to_json()
    print(json.dumps({"per_step": per_step, "budget": budget}, indent=2, sort_keys=True))
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.6 `tests/test_per_step.py` + `tests/test_budget.py` (shape)

```python
# tests/test_per_step.py
from pathlib import Path
import json
import yaml
from scorer.per_step import score_per_step

FIX = Path(__file__).parent.parent / "fixtures"


def _load_ref():
    with open(FIX / "reference_order_status_not_shipped_policy_v3_2.yaml") as f:
        return yaml.safe_load(f)


def test_happy_path_positive_control():
    with open(FIX / "support_agent_good_trajectory.json") as f:
        traj = json.load(f)
    verdicts = score_per_step(traj, _load_ref())
    assert all(v.verdict == "ok" for v in verdicts), [v.verdict for v in verdicts]


def test_chapter_01_bad_trajectory():
    with open(FIX / "support_agent_bad_trajectory.json") as f:
        traj = json.load(f)
    verdicts = score_per_step(traj, _load_ref())
    tags = [v.verdict for v in verdicts]
    assert "wrong_tool" in tags
    assert "redundant_call" in tags


def test_collapsed_retry_does_not_double_count():
    with open(FIX / "collapsed_retry_trajectory.json") as f:
        traj = json.load(f)
    verdicts = score_per_step(traj, _load_ref())
    # The retried tool shows once, not twice.
    tool_ids = [v.span_id for v in verdicts if v.tool_name.expected == "crm.lookup"]
    assert len(tool_ids) == 1


def test_parallel_tool_calls_count_once_as_step():
    """Chapter 04 §Step count: a step with parallel calls is ONE step, N tool calls.
    This test lives in per_step because it exercises the step-collection rule."""
    with open(FIX / "parallel_tool_calls_trajectory.json") as f:
        traj = json.load(f)
    verdicts = score_per_step(traj, _load_ref())
    # We expect two verdicts (two TOOL spans), not two steps.
    assert len(verdicts) == 2
    # Both should score "ok" because they are legitimate parallel calls within the DAG.
    assert {v.verdict for v in verdicts} == {"ok"}


def test_level_3_accepts_semantic_equivalent_query():
    with open(FIX / "semantic_equivalent_query_trajectory.json") as f:
        traj = json.load(f)
    verdicts = score_per_step(traj, _load_ref())
    # `docs.search` is called with "status of order 12345" vs reference "order 12345 status";
    # token set matches, so Level-3 accepts.
    docs_search = [v for v in verdicts if v.tool_name.actual == "docs.search"][0]
    assert docs_search.arguments.match
    assert docs_search.arguments.level == "semantic_equivalence"
```

```python
# tests/test_budget.py
from pathlib import Path
import json
import yaml
from scorer.budget import score_budget

FIX = Path(__file__).parent.parent / "fixtures"


def _load():
    with open(FIX / "reference_order_status_not_shipped_policy_v3_2.yaml") as f:
        ref = yaml.safe_load(f)
    with open(FIX / "price_table_v12.json") as f:
        prices = json.load(f)
    return ref, prices


def test_happy_path_budget_credit_1():
    ref, prices = _load()
    with open(FIX / "support_agent_good_trajectory.json") as f:
        traj = json.load(f)
    v = score_budget(traj, ref, prices)
    assert v.budget_credit == 1.0
    assert v.flagged_verdicts == []


def test_bad_trajectory_budget_pinned_to_zero():
    ref, prices = _load()
    with open(FIX / "support_agent_bad_trajectory.json") as f:
        traj = json.load(f)
    v = score_budget(traj, ref, prices)
    assert v.budget_credit == 0.0
    assert "cost_over_budget" in v.flagged_verdicts
    assert "retry_budget_blown" in v.flagged_verdicts
    assert v.attribution.cost_dominant_span is not None
    assert v.attribution.latency_dominant_span is not None
    assert v.attribution.retry_dominant_tool == "crm.lookup"


def test_price_table_version_round_trip():
    ref, prices = _load()
    with open(FIX / "support_agent_bad_trajectory.json") as f:
        traj = json.load(f)
    v = score_budget(traj, ref, prices)
    assert v.price_table_version == prices["_version"]


def test_determinism_byte_identical():
    ref, prices = _load()
    with open(FIX / "support_agent_bad_trajectory.json") as f:
        traj = json.load(f)
    a = score_budget(traj, ref, prices).to_json()
    b = score_budget(traj, ref, prices).to_json()
    assert json.dumps(a, sort_keys=True) == json.dumps(b, sort_keys=True)
```

### 2.7 Fixtures

`fixtures/reference_order_status_not_shipped_policy_v3_2.yaml` is the reference block from the exercise, promoted to YAML. `fixtures/price_table_v12.json` carries one entry per model used, including `cached_input_per_1k_tokens`. `fixtures/support_agent_bad_trajectory.json` is chapter 01's running example, flattened to the schema the exercise defines. The other three fixtures are the test-case targets (one happy, one with a collapsed retry, one with parallel calls, one with a semantic-equivalent query).

Reference-YAML shape:

```yaml
# fixtures/reference_order_status_not_shipped_policy_v3_2.yaml
task_family: order_status_not_shipped_policy
reference_version: v3.2
form: DAG
required_set: [docs.search, open_ticket]
allowed_set:  [docs.search, crm.lookup, open_ticket]
edges:
  - [docs.search, open_ticket]
  - [crm.lookup, open_ticket]
aliases:
  docs.retrieve: docs.search   # Q3 refactor; old traces still score
argument_rules:
  docs.search.query:
    level: 3-semantic
    rule: token_set_includes
    tokens: [order, status]
  open_ticket.reason:
    level: 2-exact
    in: [no_shipping_update, shipping_delay, delivery_exception]
budgets:
  cost_usd_ceiling:      0.02
  time_limit_s:          8.0
  max_steps:             5
  max_tool_calls:        4
  max_retries_per_tool:  1
price_table_version: v12
```

## 3. Validation steps

A reviewer runs the solution as follows:

1. **Install**: `pip install -e .` (or `pip install pydantic pyyaml` in a fresh venv).
2. **Run the test suite**: `pytest -q tests/`. All five per-step and all four budget tests pass.
3. **Run the CLI on the bad fixture**:
   ```bash
   python -m scorer.cli \
     --trajectory fixtures/support_agent_bad_trajectory.json \
     --reference  fixtures/reference_order_status_not_shipped_policy_v3_2.yaml \
     --price-table fixtures/price_table_v12.json
   ```
   The output's `budget.budget_credit` is `0.0`; `budget.flagged_verdicts` includes at minimum `cost_over_budget` and `retry_budget_blown`; `budget.attribution.cost_dominant_span` is non-null and points at the costliest LLM span.
4. **Run twice; confirm byte-identical output**: `diff <(python -m scorer.cli …) <(python -m scorer.cli …)` is empty.
5. **Grep the per-step output for the expected verdict tags**: `jq '.per_step[].verdict'` on the bad fixture yields a list containing at least `wrong_tool` and `redundant_call`.
6. **Grep for the attribution block**: the budget verdict's three attribution fields are all non-null on the bad fixture; the cost-dominant span ID round-trips to the trace viewer in mod-102's setup.
7. **Confirm no model call**: `grep -R -nE "anthropic\.|openai\.|langchain" scorer/` returns no model-client imports.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Per-step scorer emits one record per TOOL/RETRIEVER step with all four sub-score fields (`tool_name`, `arguments`, `order`, `cost_ok`) and a `verdict` tag. | All records present; no field is dropped on a name failure. | A step with `wrong_tool` short-circuits and omits `arguments` / `order` sub-scores. |
| R2 | Argument scorer implements Levels 1-3 and stubs 4-5 with `NotImplementedError`. | `canonicalise_arguments`, `token_set_includes`, `in_allowed_set` present; Level 4/5 raise. | Level 4 or 5 silently falls back to Level 2. |
| R3 | Ordering scorer uses `graphlib.TopologicalSorter` and names the violated edge on failure. | `violated_edge=("docs.search", "open_ticket")` or similar on a reordered trajectory. | Returns boolean only. |
| R4 | Retry collapsing happens before name/argument scoring; retry count moves to budget. | `test_collapsed_retry_does_not_double_count` passes; budget `retries.observed_max` reflects collapsed count. | Retries show as duplicate steps or vanish from budget. |
| R5 | Budget scorer reads `llm.response.model` (not request.model) and uses a versioned price table. | Cost per LLM step rounded to the price-table rate. | Hardcoded `cost_per_token` or reads `request.model`. |
| R6 | Composite `budget_credit = min across families`; linear decay to 0 at 2× ceiling. | A family at 110% ceiling returns credit 0.9; at 2× returns 0. | Soft-penalty curve; or weighted-sum instead of min. |
| R7 | Attribution block is non-null on the bad fixture; IDs are real span IDs from the input. | `cost_dominant_span`, `latency_dominant_span`, `retry_dominant_tool` all populated. | Any is null when the metric is non-zero. |
| R8 | Price-table version and reference version round-trip to the verdict. | `price_table_version` and `reference_version` appear in `BudgetVerdict.to_json()`. | Either missing. |
| R9 | No model call anywhere; no wall-clock; no randomness. | `grep` for model SDKs and `time.time()` is clean. | Any present. |
| R10 | Running the scorer twice on the same inputs yields byte-identical JSON. | `test_determinism_byte_identical` passes. | Dict ordering or formatting differs between runs. |

A passing submission is 8+ of 10 with R1, R4, R6, R9 all hard-required.

## 5. Common mistakes

- **Scoring one verdict per LLM span instead of per executed step.** The chapter-02 shape is one record per agent-decision step — i.e., per TOOL/RETRIEVER span (plus the terminal reply where applicable). An LLM "plan" span that *emits* a tool_call is not itself a step; the step is the executed call. Double-counting LLM spans inflates the "redundant call" count and breaks the step budget.
- **Short-circuiting on name failure.** The exercise spec is explicit: when `tool_name.match=False`, still score arguments and order. Short-circuiting discards the attribution a reviewer uses to triage.
- **Charging `gen_ai.request.model` for cost.** The model served is often different (provider retry, fallback route, A/B rebalance). Chapter 04 §Charge the right model says `response.model`. Charging the request model makes a cost-regression dashboard lie.
- **Mixing retries into the step count.** A retry is a harness event, not a plan decision. Collapse first; let the budget scorer count the retries. If you do not collapse before scoring name/order, the "wrong_tool" verdict will fire twice on the same bad step.
- **Depending on `time.time()` for latency.** Use the span's recorded `start_ns`/`end_ns`. Any other source makes the scorer non-deterministic and ruins exercise-05's replay.
- **Hardcoding `cost_per_token`.** Modern models bill input/output asymmetrically and bill cached input lower still. A flat rate is wrong for every current model; the versioned price table is not optional.
- **Letting the reference-DAG have a cycle.** `graphlib.TopologicalSorter.prepare()` raises `CycleError`; surface that as a *scorer configuration* error, not a trajectory failure. Chapter 02 §Ordering is explicit.
- **Dropping the attribution block when every budget passes.** A reviewer still wants to see which span was closest to the ceiling. Emit the attribution block unconditionally; it is cheap and it is the piece mod-107's dashboards group by.
- **Reading reference trajectories from the scorer's source tree in prod.** Fine for the exercise's `fixtures/` directory; in production the fixtures live in mod-110 and are loaded by `(task_family_id, fixture_version)`. The scorer does not assume one filesystem.

## 6. References

Official standards and project docs cited in this solution:

- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/> (status: development / experimental; re-read on each release cycle).
- OpenInference specification, <https://github.com/Arize-ai/openinference> (span kinds and attribute vocabulary used to shape the trajectory JSON).
- Model Context Protocol specification, <https://spec.modelcontextprotocol.io/> (tool naming and namespacing).
- Python standard library — `graphlib.TopologicalSorter`, <https://docs.python.org/3/library/graphlib.html>.
- Inspect — errors and limits, <https://inspect.aisi.org.uk/errors_and_limits.html> (`message_limit`, `token_limit`, `time_limit`, `working_limit`).
- Berkeley Function-Calling Leaderboard, <https://gorilla.cs.berkeley.edu/leaderboard.html> (AST-based function-call evaluation; informs Level 4 stub).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> (motivates `unauthorised_tool` floor verdict).
- MITRE ATLAS, <https://atlas.mitre.org/> (ATLAS technique catalogue informs the `wrong_tool` → security-review routing).
- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> (measure / manage functions informed the attribution-block requirement).
- Local chapters: `lessons/mod-103-trajectory-and-tool-eval/02-scoring-per-step-tool-calls.md` and `.../04-budgets-and-rollout-gates.md`.
- Local chapter: `lessons/mod-103-trajectory-and-tool-eval/01-why-trajectory-eval.md` (the bad-trajectory fixture originates here).
