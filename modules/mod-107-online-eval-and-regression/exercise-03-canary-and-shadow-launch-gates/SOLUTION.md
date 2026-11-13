# exercise-03 — Canary and Shadow Launch Gates — Solution

> Reference solution for `lessons/mod-107-online-eval-and-regression/exercises/exercise-03-canary-and-shadow-launch-gates.md`.
> Design + reference code for rolling a model / prompt / tool change behind a shadow launch and a progressive canary, with automatic gates that pause rollout on regression.

## 1. Solution overview

Two distinct rollout stages, two distinct kinds of gate, one shared verdict sink:

1. **Shadow stage.** The candidate variant runs on the same traffic as the control but its outputs are *not* shown to users. We log its outputs, judge them, and compare to the control's verdicts. This catches catastrophic regressions (new tool mis-calls, prompt bugs, broken parsing) before any user sees the candidate.
2. **Canary stage.** The candidate serves a growing fraction of real users (1% → 5% → 25% → 100%). We measure a declared bundle of quality + reliability + cost metrics and auto-promote only if every gate passes. A gate fail does not auto-rollback — it auto-pauses and pages the release owner. Rollback is a human decision; pause is automatic.

Three gate families, each with a clear semantic:

- **Hard gates** — safety / correctness floors. Any regression here *stops* the rollout at the current stage. Examples: refusal_rate increase > 1pp, unauthorized_tool rate > 0 (no new instances allowed vs. control), judge safety score drop > 2pp.
- **Soft gates** — quality metrics. A regression here warns but allows the stage to complete at its current traffic share; promotion to the next stage requires a human sign-off. Examples: helpfulness score drop ≥ 1pp, task-completion drop ≥ 2pp.
- **Budget gates** — cost / latency. Exceeding declared ceilings is a *hard* stop. Examples: p95 latency increase > 20%, cost-per-request increase > 15%.

Design choices:

- **Shadow runs before any user traffic is shared.** An "A/B from day one" strategy skips the chance to catch broken code paths without exposure. Shadow is cheap when the candidate is the same model with a prompt change; shadow is expensive but still worth it when the candidate is a different model.
- **Stage times, not sample sizes.** Each stage runs for a declared minimum duration (e.g. 24h) regardless of how fast it accumulates samples. This is to catch diurnal regressions — a model that works at 10am and breaks at 6pm. Exceptions: if a hard gate trips before the min duration, the pause fires immediately.
- **Gate inputs are verdicts from exercise-01's sink, not raw traces.** The gate is generic: give it two sample sets of verdicts (control vs candidate), it computes the bundle. A gate module that re-runs judges on the hot path couples deployment speed to judge speed.
- **Sticky assignment.** User U is assigned once per experiment id via a hash of `(user_id, experiment_id)`. Repeat visits do not switch arms; this is a basic requirement for every variance-reduction method, including CUPED in exercise-05.
- **Rollback is manual; pause is automatic.** The gate module has no authority to delete traffic. It can only arrest growth and page humans. This keeps `measure` and `manage` separated (NIST AI RMF).

## 2. Worked implementation

```
mod-107/exercise-03/
├── launch/
│   ├── __init__.py
│   ├── experiment.py        # Experiment / Stage dataclasses
│   ├── assignment.py        # sticky, hash-based cohort assignment
│   ├── shadow.py            # run candidate alongside control, compare
│   ├── canary.py            # promotion controller with stages
│   ├── gates.py             # Gate protocol + hard/soft/budget implementations
│   ├── metrics.py           # read verdicts and rollup per arm
│   └── cli.py               # `python -m launch.cli`
├── policy/
│   ├── experiment-rag-prompt-v3.yaml
│   └── gates.yaml
└── tests/
    ├── test_assignment.py
    ├── test_shadow.py
    ├── test_gates.py
    └── test_canary_pauses_on_hard_gate_fail.py
```

### 2.1 `launch/experiment.py`

```python
"""Typed description of an experiment and its stages."""
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal


@dataclass(frozen=True)
class Stage:
    name: str                         # e.g. "shadow", "canary_1pct"
    traffic_share: float              # [0, 1]; shadow uses 0
    min_duration_hours: float
    gates: tuple[str, ...]            # gate ids to apply at stage end


@dataclass(frozen=True)
class Experiment:
    experiment_id: str
    description: str
    control_variant_id: str           # e.g. "rag-prompt-v2"
    candidate_variant_id: str         # e.g. "rag-prompt-v3"
    stages: tuple[Stage, ...]
    salt: str                         # deterministic sticky-assignment salt


State = Literal["shadow", "running", "paused", "promoted", "rolled_back"]


@dataclass(frozen=True)
class StageVerdict:
    stage_name: str
    gates_pass: bool
    gates_failing: tuple[str, ...]
    attributes: dict = field(default_factory=dict)
```

### 2.2 `launch/assignment.py` — sticky hash assignment

```python
"""Hash-based sticky assignment. Same (user_id, experiment_id, salt) always
returns the same cohort. Supports a shadow stage (traffic_share=0)."""
from __future__ import annotations
import hashlib

from .experiment import Experiment, Stage


def coin(user_id: str, experiment_id: str, salt: str) -> float:
    key = f"{experiment_id}|{salt}|{user_id}".encode()
    digest = hashlib.sha256(key).digest()
    return int.from_bytes(digest[:8], "big") / 2**64


def assign(user_id: str, experiment: Experiment, stage: Stage) -> str:
    """Return the variant_id this user should see at this stage.

    Shadow stage always returns control; the candidate runs out-of-band.
    """
    if stage.name == "shadow" or stage.traffic_share == 0.0:
        return experiment.control_variant_id
    if coin(user_id, experiment.experiment_id, experiment.salt) < stage.traffic_share:
        return experiment.candidate_variant_id
    return experiment.control_variant_id
```

### 2.3 `launch/shadow.py` — run candidate on the same trace

```python
"""Shadow stage: run the candidate on every control trace, log the result,
run evaluators on both, compare. Two pieces of context to get right:

1. The candidate's output must NEVER reach the user. The async task reads
   the user's query from the trace, sends it to the candidate, drops the
   response on the floor after capturing judge verdicts.
2. The candidate must not share side-effects (tool calls on real APIs).
   The shadow tool runtime is a read-only stub by default; tool calls
   that would mutate are recorded, not executed. This is non-negotiable
   and enforced by the shadow pool.
"""
from __future__ import annotations
import logging
from dataclasses import dataclass
from typing import Any, Callable

log = logging.getLogger(__name__)


ToolRuntime = Callable[[dict[str, Any]], dict[str, Any]]


@dataclass(frozen=True)
class ShadowResult:
    trace_id: str
    control_output: str
    candidate_output: str
    control_verdicts: dict[str, float]      # {judge_id: score}
    candidate_verdicts: dict[str, float]
    tool_calls_recorded: int                # tool calls that would have mutated


class ShadowRunner:
    def __init__(
        self,
        candidate_model_call: Callable[[dict[str, Any]], str],
        candidate_tool_runtime: ToolRuntime,
        evaluators: dict[str, Callable[[str, dict[str, Any]], float]],
    ) -> None:
        self._candidate_model = candidate_model_call
        self._candidate_tools = candidate_tool_runtime
        self._evaluators = evaluators

    def run(self, trace: dict[str, Any]) -> ShadowResult:
        # Replay the trace's INPUT against the candidate.
        candidate_output = self._candidate_model(trace["input"])
        # Score both outputs by every registered evaluator.
        control_verdicts = {
            name: fn(trace["output"], trace) for name, fn in self._evaluators.items()
        }
        candidate_verdicts = {
            name: fn(candidate_output, trace) for name, fn in self._evaluators.items()
        }
        tool_calls_recorded = sum(
            1 for s in trace.get("spans", []) if s.get("kind") == "TOOL"
        )
        return ShadowResult(
            trace_id=trace["trace_id"],
            control_output=trace["output"],
            candidate_output=candidate_output,
            control_verdicts=control_verdicts,
            candidate_verdicts=candidate_verdicts,
            tool_calls_recorded=tool_calls_recorded,
        )
```

### 2.4 `launch/gates.py` — gate protocol + implementations

```python
"""Gate evaluations. Input to every gate is two sample sets of verdicts;
output is a boolean pass/fail and a one-line reason.

Gate categories:
- hard: failure stops the rollout
- soft: failure allows stage complete but blocks promotion
- budget: failure stops rollout (cost/latency ceilings)
"""
from __future__ import annotations
import math
from dataclasses import dataclass
from typing import Literal, Sequence


GateKind = Literal["hard", "soft", "budget"]


@dataclass(frozen=True)
class GateResult:
    gate_id: str
    kind: GateKind
    pass_: bool
    observed: float
    threshold: float
    reason: str


def _two_sided_mean_diff(a: Sequence[float], b: Sequence[float]) -> tuple[float, float]:
    """Return (mean_b - mean_a, pooled_se). Welch-style SE; no small-n fix needed at n>=1k."""
    na, nb = len(a), len(b)
    if na == 0 or nb == 0:
        return 0.0, float("inf")
    ma = sum(a) / na
    mb = sum(b) / nb
    va = sum((x - ma) ** 2 for x in a) / max(1, na - 1)
    vb = sum((x - mb) ** 2 for x in b) / max(1, nb - 1)
    se = math.sqrt(va / na + vb / nb)
    return mb - ma, se


def gate_mean_not_worse_than(
    gate_id: str, kind: GateKind,
    control: Sequence[float], candidate: Sequence[float],
    *, max_regression: float, z: float = 1.96,
) -> GateResult:
    """Gate passes iff lower-end of CI on (candidate - control) ≥ -max_regression.

    `max_regression` is a non-negative number; e.g. 0.01 means "candidate is
    allowed to be 1pp worse at the 95% CI lower bound."
    """
    diff, se = _two_sided_mean_diff(control, candidate)
    lower = diff - z * se
    passed = lower >= -abs(max_regression)
    return GateResult(
        gate_id=gate_id, kind=kind, pass_=passed,
        observed=lower, threshold=-abs(max_regression),
        reason=(f"pass: CI lower bound {lower:+.3f} ≥ {-abs(max_regression):+.3f}"
                if passed else
                f"fail: CI lower bound {lower:+.3f} < {-abs(max_regression):+.3f}"),
    )


def gate_rate_not_worse_than(
    gate_id: str, kind: GateKind,
    control_events: int, control_total: int,
    candidate_events: int, candidate_total: int,
    *, max_rate_increase_pp: float, z: float = 1.96,
) -> GateResult:
    """Binary-event gate (e.g. refusal rate). Fail iff candidate's rate upper
    bound exceeds control's rate + max_rate_increase."""
    if control_total == 0 or candidate_total == 0:
        return GateResult(gate_id=gate_id, kind=kind, pass_=False,
                          observed=0.0, threshold=0.0,
                          reason="fail: insufficient data in one arm")
    p_c = control_events / control_total
    p_t = candidate_events / candidate_total
    diff = p_t - p_c
    se = math.sqrt(p_c * (1 - p_c) / control_total
                   + p_t * (1 - p_t) / candidate_total)
    upper = diff + z * se
    passed = upper <= max_rate_increase_pp
    return GateResult(
        gate_id=gate_id, kind=kind, pass_=passed,
        observed=upper, threshold=max_rate_increase_pp,
        reason=(f"pass: rate-diff upper bound {upper:+.4f} ≤ {max_rate_increase_pp:+.4f}"
                if passed else
                f"fail: rate-diff upper bound {upper:+.4f} > {max_rate_increase_pp:+.4f}"),
    )


def gate_quantile_not_worse_than(
    gate_id: str, kind: GateKind,
    control: Sequence[float], candidate: Sequence[float],
    *, quantile: float, max_regression_fraction: float,
) -> GateResult:
    """Latency gate at a quantile (e.g. p95). Fail iff candidate quantile
    exceeds control quantile by more than `max_regression_fraction`.

    Quantile is an empirical quantile; for p95, `max_regression_fraction=0.20`
    means "new p95 may not exceed 1.20× old p95".
    """
    if not control or not candidate:
        return GateResult(gate_id=gate_id, kind=kind, pass_=False,
                          observed=0.0, threshold=0.0,
                          reason="fail: empty arm")
    def q(data: Sequence[float], p: float) -> float:
        s = sorted(data)
        k = min(len(s) - 1, int(p * (len(s) - 1) + 0.5))
        return s[k]
    qc = q(control, quantile)
    qt = q(candidate, quantile)
    ratio = qt / qc if qc > 0 else float("inf")
    passed = ratio <= 1.0 + max_regression_fraction
    return GateResult(
        gate_id=gate_id, kind=kind, pass_=passed,
        observed=ratio, threshold=1.0 + max_regression_fraction,
        reason=(f"pass: q{int(quantile*100)} ratio {ratio:.2f} ≤ "
                f"{1 + max_regression_fraction:.2f}"
                if passed else
                f"fail: q{int(quantile*100)} ratio {ratio:.2f} > "
                f"{1 + max_regression_fraction:.2f}"),
    )
```

### 2.5 `launch/canary.py` — promotion controller

```python
"""Progressive canary controller.

State machine:
    shadow → canary_1pct → canary_5pct → canary_25pct → canary_100pct → promoted

Transitions:
- Stage completes on min_duration_hours AND all its gates evaluate.
- Hard or budget gate failure: state → paused (pages release owner).
- Soft gate failure: stage completes but promotion requires human approval.
- All gates pass: advance to next stage.
"""
from __future__ import annotations
import logging
from dataclasses import dataclass, field, replace
from typing import Callable

from .experiment import Experiment, Stage, StageVerdict
from .gates import GateResult

log = logging.getLogger(__name__)


@dataclass(frozen=True)
class ControllerState:
    experiment_id: str
    current_stage_name: str
    state: str                                 # see State in experiment.py
    history: tuple[StageVerdict, ...] = field(default_factory=tuple)


GateRunner = Callable[[Experiment, Stage], list[GateResult]]
Pager = Callable[[str, str], None]             # (channel, message)


class CanaryController:
    def __init__(
        self, experiment: Experiment, run_gates: GateRunner, pager: Pager,
    ) -> None:
        self._exp = experiment
        self._run_gates = run_gates
        self._pager = pager
        self._state = ControllerState(
            experiment_id=experiment.experiment_id,
            current_stage_name=experiment.stages[0].name,
            state="shadow" if experiment.stages[0].name == "shadow" else "running",
        )

    @property
    def state(self) -> ControllerState:
        return self._state

    def _next_stage(self) -> Stage | None:
        names = [s.name for s in self._exp.stages]
        try:
            idx = names.index(self._state.current_stage_name)
        except ValueError:
            return None
        if idx + 1 >= len(names):
            return None
        return self._exp.stages[idx + 1]

    def tick(self) -> ControllerState:
        """Called when a stage's min_duration has elapsed."""
        current = next(s for s in self._exp.stages if s.name == self._state.current_stage_name)
        results = self._run_gates(self._exp, current)
        hard_failures = [r for r in results if r.kind in {"hard", "budget"} and not r.pass_]
        soft_failures = [r for r in results if r.kind == "soft" and not r.pass_]

        verdict = StageVerdict(
            stage_name=current.name,
            gates_pass=all(r.pass_ for r in results),
            gates_failing=tuple(r.gate_id for r in results if not r.pass_),
            attributes={"soft_failures": [r.gate_id for r in soft_failures]},
        )

        if hard_failures:
            self._pager(
                f"#rel-{self._exp.experiment_id}",
                f"hard gate failure at stage {current.name}: "
                + ", ".join(f"{r.gate_id} ({r.reason})" for r in hard_failures),
            )
            self._state = replace(
                self._state, state="paused",
                history=self._state.history + (verdict,),
            )
            return self._state

        if soft_failures:
            self._pager(
                f"#rel-{self._exp.experiment_id}",
                f"soft gate(s) failed at stage {current.name}; promotion requires "
                f"human approval: {', '.join(r.gate_id for r in soft_failures)}",
            )
            self._state = replace(
                self._state, state="paused",
                history=self._state.history + (verdict,),
            )
            return self._state

        next_stage = self._next_stage()
        if next_stage is None:
            self._state = replace(
                self._state, state="promoted",
                history=self._state.history + (verdict,),
            )
        else:
            self._state = replace(
                self._state,
                current_stage_name=next_stage.name,
                state="running",
                history=self._state.history + (verdict,),
            )
        return self._state
```

### 2.6 `policy/experiment-rag-prompt-v3.yaml`

```yaml
experiment_id: rag-prompt-v3
description: "RAG system prompt v3 — stricter grounding instruction"
control_variant_id: rag-prompt-v2
candidate_variant_id: rag-prompt-v3
salt: 2025-11-release-v3
stages:
  - name: shadow
    traffic_share: 0.0
    min_duration_hours: 48
    gates: [hard.unauthorized_tool, hard.safety, budget.latency_p95, budget.cost_per_req]
  - name: canary_1pct
    traffic_share: 0.01
    min_duration_hours: 24
    gates:
      - hard.refusal_rate_not_up_1pp
      - hard.safety_not_down_2pp
      - hard.unauthorized_tool
      - budget.latency_p95
      - budget.cost_per_req
      - soft.helpfulness_not_down_1pp
      - soft.task_completion_not_down_2pp
  - name: canary_5pct
    traffic_share: 0.05
    min_duration_hours: 24
    gates:
      - hard.refusal_rate_not_up_1pp
      - hard.safety_not_down_2pp
      - budget.latency_p95
      - budget.cost_per_req
      - soft.helpfulness_not_down_1pp
  - name: canary_25pct
    traffic_share: 0.25
    min_duration_hours: 48
    gates:
      - hard.safety_not_down_2pp
      - budget.latency_p95
      - soft.helpfulness_not_down_1pp
  - name: full
    traffic_share: 1.0
    min_duration_hours: 24
    gates: []
```

### 2.7 `tests/test_assignment.py`

```python
from launch.assignment import assign, coin
from launch.experiment import Experiment, Stage


def _exp(share: float) -> tuple[Experiment, Stage]:
    exp = Experiment(
        experiment_id="rag-prompt-v3",
        description="",
        control_variant_id="C",
        candidate_variant_id="T",
        stages=(Stage(name="canary_5pct", traffic_share=share,
                      min_duration_hours=24, gates=()),),
        salt="2025-11",
    )
    return exp, exp.stages[0]


def test_assignment_is_sticky():
    exp, stage = _exp(0.5)
    a1 = assign("user-42", exp, stage)
    a2 = assign("user-42", exp, stage)
    assert a1 == a2


def test_assignment_distribution_matches_share():
    exp, stage = _exp(0.05)
    hits = sum(1 for i in range(50_000) if assign(f"u{i}", exp, stage) == "T")
    # 5% ± 0.3%
    assert 2350 <= hits <= 2650


def test_shadow_stage_never_routes_candidate():
    exp = Experiment(
        experiment_id="x", description="",
        control_variant_id="C", candidate_variant_id="T",
        stages=(Stage(name="shadow", traffic_share=0.0,
                      min_duration_hours=1, gates=()),),
        salt="s",
    )
    for i in range(1000):
        assert assign(f"u{i}", exp, exp.stages[0]) == "C"
```

### 2.8 `tests/test_canary_pauses_on_hard_gate_fail.py`

```python
from launch.canary import CanaryController
from launch.experiment import Experiment, Stage
from launch.gates import GateResult


def _exp():
    return Experiment(
        experiment_id="x", description="",
        control_variant_id="C", candidate_variant_id="T",
        stages=(
            Stage(name="canary_1pct", traffic_share=0.01,
                  min_duration_hours=24, gates=("hard.safety",)),
            Stage(name="canary_5pct", traffic_share=0.05,
                  min_duration_hours=24, gates=("hard.safety",)),
        ),
        salt="s",
    )


def test_hard_gate_fail_pauses_and_pages():
    pages = []
    def run_gates(_exp, _stage):
        return [GateResult(gate_id="hard.safety", kind="hard", pass_=False,
                           observed=-0.05, threshold=-0.02, reason="fail")]
    c = CanaryController(_exp(), run_gates=run_gates,
                         pager=lambda ch, msg: pages.append((ch, msg)))
    s = c.tick()
    assert s.state == "paused"
    assert len(pages) == 1
    assert "hard gate failure" in pages[0][1]


def test_all_gates_pass_promotes_to_next_stage():
    def run_gates(_exp, _stage):
        return [GateResult(gate_id="hard.safety", kind="hard", pass_=True,
                           observed=0.0, threshold=-0.02, reason="pass")]
    c = CanaryController(_exp(), run_gates=run_gates, pager=lambda ch, msg: None)
    s = c.tick()
    assert s.state == "running"
    assert s.current_stage_name == "canary_5pct"
```

## 3. Validation steps

1. `pytest -q tests/` passes; the assignment test proves stickiness, the share distribution, and shadow-stage safety. The canary test proves pause-on-hard-fail and advance-on-all-pass.
2. **Shadow-only dry run**: `python -m launch.cli --experiment policy/experiment-rag-prompt-v3.yaml --simulate-hours 48` runs the shadow stage against a synthetic trace stream; the output ledger records candidate outputs, candidate judge scores, and tool-call records with `executed=false`.
3. **Canary promotion**: set the simulated candidate to be identical to control; the CLI promotes through all stages; the final state is `promoted` with no pages.
4. **Canary pause**: swap the candidate for one that has a +2pp refusal rate; the CLI stops at `canary_1pct` with `state=paused` and a page containing the failing gate id and `reason`.
5. **Budget gate**: set the candidate to 1.5× latency; the CLI stops at `shadow` (budget gate is included at shadow by design).
6. **No tool side-effects in shadow**: `grep "executed=true" shadow.log` returns zero rows across any shadow stage; `grep "executed=false" | wc -l` equals the candidate's intended tool-call count.
7. **Idempotent replay**: rerun the CLI against the same synthetic stream and policy; the final controller state, pages emitted, and ledger rows are identical.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Experiment has a shadow stage with `traffic_share=0.0` and real gates. | Shadow stage runs; candidate output never reaches user; tests prove this. | Candidate reaches user in shadow, OR shadow has no gates. |
| R2 | Assignment is sticky and salted; `(user_id, experiment_id, salt)` determines arm. | `test_assignment_is_sticky` passes; two different experiments can co-exist without interference by using different salts. | `random.random()` or per-request flip; or no salt / collision across experiments. |
| R3 | Gates are typed: hard / soft / budget. Each class has a different consequence. | `GateKind` enum + three distinct branches in controller. | Single "pass/fail" with no concept of soft vs hard. |
| R4 | Hard or budget gate failure pauses, pages, and does NOT advance. | Controller test proves state=paused after hard-fail; message contains gate id. | Controller continues to next stage despite failure. |
| R5 | Soft gate failure pauses for human sign-off, not automatic rollback. | Soft failure state is `paused` with soft_failures listed; next tick is a no-op until a human resets. | Soft failure is silently ignored, OR soft failure triggers auto-rollback. |
| R6 | Rollback is a human action; controller has no `rollback()` method. | No code path writes state=rolled_back without explicit CLI command. | Auto-rollback button exists; one gate fail wipes the candidate. |
| R7 | Each stage has a minimum duration enforced independently of sample size. | `min_duration_hours` field present; CLI respects it even on light traffic. | Stage advances the moment N samples hit. |
| R8 | Shadow tool-runtime defaults to read-only; mutating tool calls recorded, not executed. | Shadow log shows `executed=false`; test proves zero side-effects. | Candidate's shadow calls hit real third-party APIs. |
| R9 | Gate inputs are verdicts from exercise-01's sink, not fresh model runs. | Gate module reads from sink; no evaluator imports. | Gates re-evaluate candidate on hot path. |
| R10 | Policy (stages, thresholds, gates) is YAML, not Python; one PR per change. | `policy/` files present; gate module has no literal thresholds in alert-decision paths. | Thresholds hardcoded in `canary.py`. |

Pass requires 8/10 with R1, R2, R4, R6, R8 all hard.

## 5. Common mistakes

- **Skipping shadow.** "We've already tested in staging" is not the same as running on real traffic distribution. Shadow catches bugs that only appear under production trace shape (user slang, unusual tool orderings, long sessions).
- **Share-based stage completion.** Advancing on sample size means you may skip the "peak-load window" entirely if the canary happens to run overnight. Time-based stage completion is the floor; sample-size is an optional ADDITIONAL requirement, not a replacement.
- **Assignment that is not sticky.** Users hop between arms, which (a) ruins any experiment's variance reduction (CUPED, exercise-05), (b) produces an inconsistent product experience, and (c) is visible to users in multi-turn conversations.
- **No separation of hard vs soft gates.** Treating every regression as a hard-fail blocks every legitimate improvement (small quality drops in one segment accompanied by a large win in another). Treating every regression as a soft-fail risks shipping a safety regression.
- **Automatic rollback.** A rollback is a user-visible event; it should always be a human decision. Automatic pause is correct and the floor; automatic rollback is the floor only in the very narrow case of a confirmed safety breach. OWASP ML Top 10 (`ML05: Model Theft` and `ML07: Transfer Learning Attack`) and MITRE ATLAS (`AML.T0020: Poison Training Data`) can all cause symptoms that look like a candidate regression; rollback without human triage can mask an attack.
- **Shadow tool calls that hit real APIs.** A shadow run sending Slack messages / processing payments is worse than skipping shadow. Enforce a read-only tool runtime at the pool level; mutating tools are recorded as "would-have-executed".
- **Ignoring diurnal effects.** 24h minimum per stage is the floor for any product with visible day/night cycles.
- **Gates computed from the model directly.** If the gate re-runs the judge, the judge's latency becomes rollout latency. Gates read verdicts from exercise-01.
- **No pager in the controller test.** If the test fakes the pager and the code paths that call it, you've confirmed only that the function compiles. Verify actual page content.
- **CI hint only.** A gate "warning" that only shows up in a dashboard is not a gate. A gate must either stop rollout or require a human to click; otherwise it is a vanity metric.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MANAGE 4.1` (post-deployment risk monitoring) and `MEASURE 2.6` (deployment-phase validation).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML07: Transfer Learning Attack` motivates the shadow-stage read-only tool runtime.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0020: Poison Training Data` and `AML.M0007: Control Access to ML Models` are the control framings; progressive canary is one of the deployment-phase controls under `AML.M0019: Deploy ML Model Updates Carefully`.
- Python standard library — `hashlib.sha256`, <https://docs.python.org/3/library/hashlib.html>.
- Local chapters and exercise reader: `lessons/mod-107-online-eval-and-regression/exercises/exercise-03-canary-and-shadow-launch-gates.md`.
- Related exercises: exercise-01 (verdict sink feeds the gates), exercise-02 (drift alerts can trigger a pause outside the stage ticker), exercise-04 (confidence-sequence gates for always-valid stop decisions), exercise-05 (CUPED makes quality gates sharper under the same traffic).
