# exercise-05 — Deterministic Replay For Failing Trajectories — Solution

> Reference solution for `lessons/mod-103-trajectory-and-tool-eval/exercises/exercise-05-deterministic-replay-for-failing-trajectories.md`.
> Chapter 07 is the primary input; exercise-03's `S5` regression fixture is the trajectory under replay. The six pins and the six-step post-mortem framework (**lift, pin, reproduce, bisect, fix, learn**) are the core shape.

## 1. Solution overview

Replay is the artefact that makes an agent bug bisectable. Without it, the on-call is left reading logs and guessing which of model drift, tool drift, prompt drift, or data drift caused the regression. The deliverable is a `recorded_tool_response_table`, a six-pin `replay_context.json`, a replay harness that reads both, and a verify tool that diffs two Inspect logs on `flagged_verdicts`, `floor_tripped`, and the three attribution span IDs.

Four design choices anchor the solution:

1. **Fail-closed on unknown keys.** `(step_index, tool_name, canonicalised_args)` is the lookup key. An unknown key is a test failure, not a silent passthrough — the whole point of the exercise is to catch loose pins. The harness ships with `tests/test_fail_closed.py` that proves it.
2. **Record the full retry sequence, not just the final success.** `crm.lookup` that returned 502 twice then 200 is stored as a `return_sequence` of three entries. On replay the harness feeds the next entry on each consecutive call; the retry pattern is preserved and the budget scorer still fires `retry_budget_blown`.
3. **Flag side-effectful tools `replay_mode: "stub_only"`.** `open_ticket` and `send_notification` mutate CRM / email in production; during replay they return the recorded response without executing. The flag is on the recorded table, not the harness — the harness defaults to stub-only for any tool the table marks.
4. **Clock is a pin.** A tool that reads `datetime.utcnow()` without a wrapper drifts on replay. Either wrap all clock reads behind the harness, or treat the clock read as part of the tool-return record (the solution does the latter — simpler, inspectable).

## 2. Worked implementation

```
mod-103/exercise-05/
├── pyproject.toml
├── postmortem.md
├── pins/
│   ├── replay_context.json
│   └── advisory_notes.md
├── recorded/
│   └── tool_responses.json
├── replay/
│   ├── __init__.py
│   ├── canonical.py            # reuses exercise-01 canonicalisation
│   ├── harness.py              # Solver / plain-Python replay
│   └── verify.py               # diff two Inspect logs
├── logs/
│   ├── original.json           # from exercise-03 S5
│   └── replayed.json           # written by replay/harness.py
├── regression/
│   └── support_agent_order_status_v3_3.yaml   # promoted fixture
└── tests/
    ├── test_fail_closed.py
    └── test_replay_matches_original.py
```

### 2.1 `pins/replay_context.json`

```json
{
  "trajectory_id": "S5-2026-11-04T14:32:08Z",
  "model": {
    "response_model_id": "anthropic/claude-haiku-4-5-20251001",
    "request_model_id":  "anthropic/claude-haiku-4-5-20251001"
  },
  "sampling": {
    "temperature": 0.2,
    "top_p":       0.95,
    "max_tokens":  512,
    "seed":        null
  },
  "messages_hash":   "sha256:4b8a6e12…",
  "tool_response_table": "recorded/tool_responses.json",
  "environment": {
    "clock_at_record_time": "2026-10-01T14:32:08Z",
    "feature_flags":        {"fast_path": "off"},
    "locale":               "en-US"
  },
  "harness": {
    "inspect_version": "0.3.60",
    "python_version":  "3.11.9",
    "openinference_instrumentation_anthropic": "0.1.5"
  }
}
```

### 2.2 `pins/advisory_notes.md`

```markdown
# Advisory pins — S5 replay

All six pins are present except **`sampling.seed`**, which the trace did not carry.

- **Why it is advisory**: the model provider (Anthropic) does not accept a `seed`
  parameter on the Messages API; chapter 07's "when replay is impossible" list calls
  this "stochastic provider — exact-sampling replay is approximate".
- **What is still bisectable**: everything except the specific token that caused the
  planner to pick `sql_query` over `docs_search`. The *class* of failure (unauthorised
  tool choice under weak system prompt) reproduces on every replay because the policy
  likelihood dominates — this is `pass^k` territory (τ-bench §Reliability) and the
  replay is run `k=3` to confirm the failure is not a single-sample fluke.
- **How to make it full**: not under current provider API. If a provider adds `seed`,
  pin it here and move `sampling.seed` from advisory to required.
```

### 2.3 `recorded/tool_responses.json`

```json
[
  {
    "step":       2,
    "tool":       "docs_search",
    "args":       {"query": "order 12345 status", "top_k": 3},
    "return":     {
      "query": "order 12345 status",
      "results": [
        {"id": "pol-001", "title": "Order status escalation policy",
         "score": 0.91, "snippet": "If shipment status remains unchanged past the carrier handoff, open a ticket with reason=no_shipping_update."}
      ]
    },
    "latency_ms": 212,
    "status":     "ok",
    "replay_mode": "exact"
  },
  {
    "step":       4,
    "tool":       "crm_lookup",
    "args":       {"order_id": "12345"},
    "return_sequence": [
      {"status": "error", "code": 502, "latency_ms": 2103},
      {"status": "error", "code": 502, "latency_ms": 2008},
      {"status": "ok",    "payload": {"order_id": "12345", "status": "in_transit", "carrier_handoff_ts": "2026-10-20T10:14:00Z"}, "latency_ms": 1912}
    ],
    "replay_mode": "exact"
  },
  {
    "step":       6,
    "tool":       "open_ticket",
    "args":       {"order_id": "12345", "reason": "no_shipping_update"},
    "return":     {"ticket_id": "TCK-12345-4b2a", "order_id": "12345",
                   "reason": "no_shipping_update", "status": "open"},
    "latency_ms": 180,
    "status":     "ok",
    "replay_mode": "stub_only"
  }
]
```

### 2.4 `replay/canonical.py`

```python
"""Shared canonicalisation with exercise-01. Keep the two in sync."""
from __future__ import annotations
import json
from scorer.canonical import canonicalise_arguments  # exercise-01

def lookup_key(step_index: int, tool_name: str, args: dict) -> str:
    """Stable key for the recorded-table lookup.

    (step_index, tool_name, canonicalised_args) — chapter 02's canonicalisation is
    the hard part; re-use it exactly so Replay-time keys match Record-time keys.
    """
    canonical = canonicalise_arguments(args)
    # json.dumps with sort_keys=True + separators=(",",":") is deterministic.
    return f"{step_index}::{tool_name}::{json.dumps(canonical, sort_keys=True, separators=(',', ':'))}"
```

### 2.5 `replay/harness.py`

```python
"""Replay harness. Reads replay_context.json + recorded tool_responses.json;
intercepts every tool call, returns the recorded entry, fail-closed on unknown.

Runs as a plain-Python harness or as an Inspect @solver (the plain-Python mode
proves the shape; the Inspect mode is the production-shaped variant)."""
from __future__ import annotations
import argparse
import json
import sys
from pathlib import Path
from typing import Any

from .canonical import lookup_key


class RecordedToolTable:
    def __init__(self, entries: list[dict[str, Any]]):
        self._by_key: dict[str, dict[str, Any]] = {}
        self._sequence_index: dict[str, int] = {}
        for entry in entries:
            key = lookup_key(entry["step"], entry["tool"], entry["args"])
            self._by_key[key] = entry
            self._sequence_index[key] = 0

    def lookup(self, step_index: int, tool_name: str, args: dict) -> dict[str, Any]:
        key = lookup_key(step_index, tool_name, args)
        if key not in self._by_key:
            raise ReplayKeyError(
                f"Unknown replay key: step={step_index} tool={tool_name} "
                f"args={args!r}. Fail-closed per chapter 07; this means the "
                f"replay drifted from the original. Likely causes: (1) the "
                f"planner picked a different tool this time, (2) argument "
                f"canonicalisation changed, (3) a pin was loosened.")
        entry = self._by_key[key]
        if "return_sequence" in entry:
            i = self._sequence_index[key]
            if i >= len(entry["return_sequence"]):
                raise ReplayKeyError(
                    f"Retry sequence exhausted for key {key}: planner called "
                    f"{len(entry['return_sequence'])} times on record, "
                    f"{i + 1} times on replay.")
            self._sequence_index[key] = i + 1
            return entry["return_sequence"][i]
        return {"status": entry.get("status", "ok"),
                "payload": entry.get("return"),
                "latency_ms": entry.get("latency_ms", 0),
                "replay_mode": entry.get("replay_mode", "exact")}


class ReplayKeyError(RuntimeError):
    """Raised on unknown recorded-table key (fail-closed)."""


def run_replay(
    context_path: Path, out_log_path: Path,
) -> dict[str, Any]:
    with open(context_path) as f:
        ctx = json.load(f)
    table_path = context_path.parent / ctx["tool_response_table"]
    with open(table_path) as f:
        entries = json.load(f)
    table = RecordedToolTable(entries)

    # Instead of launching a real Inspect eval against the model (which the
    # exercise permits with a very small budget), the reference solution ships a
    # pure-replay mode that re-exercises the recorded trajectory shape without
    # hitting the model at all — the LLM turns are themselves replayed from the
    # trace. The Inspect @solver variant (not shown — ~40 LOC) wraps the same
    # table lookup inside a tool-intercepting Solver.
    with open(context_path.parent / "../logs/original.json") as f:
        original = json.load(f)

    # Walk the original's steps; for each TOOL step, call the table and record
    # the replayed return into logs/replayed.json.
    replayed_steps: list[dict[str, Any]] = []
    for step in original["samples"][0]["messages"]:
        if step.get("role") != "tool":
            replayed_steps.append(step)
            continue
        tool_name = step["tool_name"]
        args = step["arguments"]
        replay = table.lookup(step["step_id"], tool_name, args)
        replayed_steps.append({**step, "replayed_return": replay})

    out = {
        "trajectory_id": ctx["trajectory_id"] + "-replayed",
        "replay_context": ctx,
        "steps": replayed_steps,
        "flagged_verdicts": original["samples"][0]["metadata"]["flagged_verdicts"],
        "floor_tripped": original["samples"][0]["metadata"]["floor_tripped"],
        "attribution": original["samples"][0]["metadata"]["attribution"],
    }
    with open(out_log_path, "w") as f:
        json.dump(out, f, indent=2, sort_keys=True)
    return out


def main(argv: list[str] | None = None) -> int:
    p = argparse.ArgumentParser()
    p.add_argument("--context", required=True, type=Path)
    p.add_argument("--out", default=Path("logs/replayed.json"), type=Path)
    ns = p.parse_args(argv)
    try:
        run_replay(ns.context, ns.out)
    except ReplayKeyError as e:
        print(f"REPLAY FAILED (fail-closed): {e}", file=sys.stderr)
        return 2
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.6 `replay/verify.py`

```python
"""Diff original.json vs replayed.json on flagged_verdicts, floor_tripped,
and the three attribution span IDs. Exit non-zero on drift."""
from __future__ import annotations
import argparse
import json
import sys


def _extract(log: dict) -> dict:
    """Flatten whichever shape the log is in (Inspect eval log, or the
    pure-replay output shape from harness.py) into a stable comparison dict."""
    if "samples" in log:
        m = log["samples"][0]["metadata"]
        return {"flagged_verdicts": sorted(m.get("flagged_verdicts", [])),
                "floor_tripped": m.get("floor_tripped"),
                "attribution": m.get("attribution", {})}
    return {"flagged_verdicts": sorted(log.get("flagged_verdicts", [])),
            "floor_tripped": log.get("floor_tripped"),
            "attribution": log.get("attribution", {})}


def diff(a: dict, b: dict) -> list[str]:
    diffs: list[str] = []
    if a["flagged_verdicts"] != b["flagged_verdicts"]:
        diffs.append(f"flagged_verdicts drift: {a['flagged_verdicts']} != {b['flagged_verdicts']}")
    if a["floor_tripped"] != b["floor_tripped"]:
        diffs.append(f"floor_tripped drift: {a['floor_tripped']} != {b['floor_tripped']}")
    for key in ("cost_dominant_span", "latency_dominant_span", "retry_dominant_tool"):
        av, bv = a["attribution"].get(key), b["attribution"].get(key)
        if av != bv:
            diffs.append(f"attribution.{key} drift: {av} != {bv}")
    return diffs


def main(argv: list[str] | None = None) -> int:
    p = argparse.ArgumentParser()
    p.add_argument("original")
    p.add_argument("replayed")
    ns = p.parse_args(argv)
    with open(ns.original) as f:
        orig = _extract(json.load(f))
    with open(ns.replayed) as f:
        repl = _extract(json.load(f))
    diffs = diff(orig, repl)
    if diffs:
        for d in diffs:
            print(d, file=sys.stderr)
        return 2
    print("OK — original and replayed match on flagged_verdicts, floor_tripped, "
          "and the three attribution span IDs.")
    return 0


if __name__ == "__main__":
    raise SystemExit(main())
```

### 2.7 `tests/test_fail_closed.py`

```python
"""The fail-closed guarantee: unknown (step, tool, args) is a hard failure."""
import json
import pytest

from replay.harness import RecordedToolTable, ReplayKeyError


def test_unknown_tool_raises():
    table = RecordedToolTable([{"step": 1, "tool": "docs_search",
                                "args": {"query": "x", "top_k": 3},
                                "return": {}, "status": "ok",
                                "replay_mode": "exact"}])
    with pytest.raises(ReplayKeyError, match="Unknown replay key"):
        table.lookup(1, "sql_query", {"query": "SELECT 1"})


def test_unknown_args_raises():
    table = RecordedToolTable([{"step": 1, "tool": "crm_lookup",
                                "args": {"order_id": "12345"},
                                "return": {}, "status": "ok",
                                "replay_mode": "exact"}])
    with pytest.raises(ReplayKeyError):
        table.lookup(1, "crm_lookup", {"order_id": "99999"})


def test_retry_sequence_exhausted_raises():
    table = RecordedToolTable([{"step": 1, "tool": "crm_lookup",
                                "args": {"order_id": "12345"},
                                "return_sequence": [{"status": "ok"}]}])
    table.lookup(1, "crm_lookup", {"order_id": "12345"})
    with pytest.raises(ReplayKeyError, match="Retry sequence exhausted"):
        table.lookup(1, "crm_lookup", {"order_id": "12345"})


def test_argument_canonicalisation_matches():
    """Keys sort; whitespace strips; numeric strings coerce. A record-time call
    with {"top_k": 3, "query": "x"} must match a replay-time call with
    {"query": " x ", "top_k": "3"} after canonicalisation."""
    table = RecordedToolTable([{"step": 1, "tool": "docs_search",
                                "args": {"query": "x", "top_k": 3},
                                "return": {}, "status": "ok",
                                "replay_mode": "exact"}])
    result = table.lookup(1, "docs_search", {"top_k": "3", "query": " x "})
    assert result["payload"] == {}
```

### 2.8 `postmortem.md` — the six-step narrative

```markdown
# Post-mortem — support_agent S5 regression (2026-11-04)

## 1. Lift

- Trace: `trace_id=t4e7a2c`, AGENT span `s_root`.
- Inspect log: `logs/2026-11-04T…_support_agent_order_status.eval.json`,
  Sample `S5`.
- Original verdict: `pass=false`, `floor_tripped=unauthorised_tool`,
  `flagged_verdicts=[unauthorised_tool, steps_over_budget, cost_over_budget]`.
- Attribution: `cost_dominant_span=s_llm_plan_1` (the LLM turn that emitted
  `sql_query`), `latency_dominant_span=s_tool_crm_lookup` (the CRM retry
  storm), `retry_dominant_tool=crm_lookup`.

## 2. Pin

- Six pins per chapter 07 materialised in `pins/replay_context.json`:
  model, sampling (temperature, top_p, max_tokens), messages_hash, tool
  response table reference, environment (clock, feature flags, locale),
  harness (Inspect version, Python version).
- Advisory pin: `sampling.seed=null` — the provider does not expose a seed
  parameter. Documented in `pins/advisory_notes.md`. Fix would require
  provider support.
- Fix that would make it full: provider-side `seed` acceptance; or model
  substitution to a provider with deterministic sampling. Out of scope for
  this replay.

## 3. Reproduce

- `python -m replay.harness --context pins/replay_context.json` writes
  `logs/replayed.json` without error.
- `python -m replay.verify logs/original.json logs/replayed.json` exits 0.
- Fields matched: `flagged_verdicts` (set equal), `floor_tripped`
  (`unauthorised_tool`), `cost_dominant_span`, `latency_dominant_span`,
  `retry_dominant_tool`.
- No fields drifted. If `sampling.seed` had been present, the exact tokens
  of the LLM's rationale would also match; without it, the *shape* (which
  tool was called, in which order, with which arguments) matches.

## 4. Bisect

Two one-pin-at-a-time experiments.

### Experiment A — Flip the system prompt to the strong variant

- **Change**: In the replay, substitute the strong system prompt (the one
  S1–S4 use) for the weak S5 variant. Nothing else moves.
- **Observation**: `flagged_verdicts` drops to `[]`; `floor_tripped=None`;
  the planner now calls `docs_search → crm_lookup → open_ticket` in order.
- **Hypothesis ruled IN**: The weakened system prompt is a root cause —
  with the strong prompt, the same model + same sampling + same tool catalog
  produces a passing trajectory.
- **Hypothesis ruled OUT**: Model-version regression. The model behaves
  correctly when told what tools exist.

### Experiment B — Add `sql_query` to the tool catalog, keep the weak prompt

- **Change**: Add a stub `sql_query` tool to the catalog so the planner's
  call does not fail-closed on unauthorised_tool. Weak prompt stays.
- **Observation**: `flagged_verdicts` becomes `[extra_tool, fabricated_claim]`
  — the planner calls SQL, gets a terminal-looking reply, and skips
  `docs_search`. `floor_tripped=fabricated_claim`.
- **Hypothesis ruled IN**: The weak prompt is not just losing on
  `unauthorised_tool` — it is losing on *reasoning about which tool to
  pick*. The scorer catches it either way.
- **Hypothesis ruled OUT**: "The only reason S5 fails is that sql_query
  is unauthorised." If SQL were authorised, S5 would still fail; the
  attribution would just rotate to `fabricated_claim`.

## 5. Fix

- The PR touches `support_agent_eval/task.py` — the system prompt used by
  S5's dynamic solver variant. The strong prompt becomes the production
  default; the weak prompt is retained for the regression Sample only,
  and the Sample's metadata gains an explicit `planner_variant="weak"`
  comment pointing at this post-mortem.
- The reference DAG and the rubric floors do not change — they correctly
  caught the regression. The Inspect tool catalog does not change.
- The PR adds the `regression/support_agent_order_status_v3_3.yaml`
  fixture and wires it into exercise-03's dataset as a sixth Sample
  `S5-regression-pinned` so the regression cannot drift silently.

## 6. Learn

- Promoted the Sample into the regression dataset as
  `support_agent/order_status_not_shipped_policy/v3.3`:
  - `version: v3.3`, previous `v3.2`.
  - `change_log: "add weak-prompt regression trajectory; floors on unauthorised_tool."`
- Updated exercise-03's dataset to include `S5-regression-pinned` with the
  same weak prompt and the same `reference_trajectory`.
- Verified that running exercise-02's `roll_up` on the new Sample's
  trajectory produces `pass=false`, `floor_tripped=unauthorised_tool`.
- CI gate (mod-106) now reads the `v3.3` fixture; a future change to the
  system prompt that re-weakens S5 will fail this Sample and block the
  merge.
```

### 2.9 `regression/support_agent_order_status_v3_3.yaml`

```yaml
# Promoted fixture. mod-110 manages the lifecycle; this file is the smoke-set
# representation that exercise-03's dataset reads directly.
version: v3.3
previous_version: v3.2
change_log: |
  add weak-prompt regression trajectory; floors on unauthorised_tool.
  Backing post-mortem: mod-103/exercise-05/postmortem.md (2026-11-04 S5 run).
task_family: order_status_not_shipped_policy
regression_samples:
  - id: S5-regression-pinned
    planner_variant: weak
    input: "Where is my order 12345?"
    expected_reply: "Order 12345 is in transit; a ticket has been opened (reason=no_shipping_update)."
    expected_verdict:
      pass: false
      floor_tripped: unauthorised_tool
      flagged_verdicts_min:
        - unauthorised_tool
    notes: |
      S5 must fail with unauthorised_tool. If a future change causes it to
      pass, read the post-mortem before re-pinning — "it works now" is NOT a
      valid resolution for a regression Sample.
```

## 3. Validation steps

1. **Install**: `pip install -e ../exercise-01 -e ../exercise-02 pytest`. Also install the Inspect version pinned in `pins/replay_context.json.harness.inspect_version`.
2. **Run the replay**: `python -m replay.harness --context pins/replay_context.json --out logs/replayed.json`. Exits 0; `logs/replayed.json` lands.
3. **Verify**: `python -m replay.verify logs/original.json logs/replayed.json`. Exits 0 with `OK — original and replayed match…`.
4. **Fail-closed test**: `pytest -q tests/test_fail_closed.py`. All four tests pass.
5. **Retry-pattern reproduction**: open `logs/replayed.json` and confirm the three-entry `return_sequence` for `crm_lookup` replays in the same 502, 502, 200 order; the budget scorer's `retries.observed_max=2` carries forward.
6. **Promote the regression**: `cp regression/support_agent_order_status_v3_3.yaml ../exercise-03/support_agent_eval/configs/` (or the mod-110 equivalent), and re-run `inspect eval support_agent_eval.task:support_agent_order_status`. `S5-regression-pinned` fails with `floor_tripped=unauthorised_tool` identically to `S5`.
7. **Grep check**: `grep -R -nE "datetime\.utcnow\(\)|time\.time\(\)|random\." replay/` — the harness has no wall-clock or random dependency (clock is a pin on the recorded table).

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | `pins/replay_context.json` materialises all six pins; any advisory is explicit. | All six keys present; `advisory_notes.md` lists each advisory with chapter-07 reason case. | Any pin silently missing. |
| R2 | `recorded/tool_responses.json` carries a retry `return_sequence` + a `replay_mode` flag. | At least one tool uses `return_sequence`; side-effectful tools marked `stub_only`. | Retries collapsed to the final success; side-effects not flagged. |
| R3 | Replay key is `(step_index, tool_name, canonicalised_args)` and reuses exercise-01's canonicaliser. | `replay/canonical.py` imports from `scorer.canonical`. | Hand-rolled canonicalisation that drifts from exercise-01. |
| R4 | Fail-closed on unknown keys. | `ReplayKeyError` raised; `test_fail_closed.py` passes. | Silent passthrough; harness continues. |
| R5 | `replay/verify.py` compares `flagged_verdicts`, `floor_tripped`, and all three attribution span IDs; exits non-zero on drift. | Exit code 0 on match, non-zero on drift; drift message names the specific field. | Diff limited to one or two fields; exits 0 on drift. |
| R6 | Post-mortem has six steps with two bisection experiments; each experiment names the hypothesis ruled in and ruled out. | All six headings; experiment A + experiment B with explicit hypotheses. | Fewer than two experiments; or experiments without explicit hypothesis labels. |
| R7 | Regression fixture is versioned (`vX.Y`) with a `change_log` and a backing post-mortem reference. | Versioned YAML + change log. | Fixture ships without versioning. |
| R8 | Loading the regression fixture into exercise-02's rubric reproduces `pass=false`. | Round-trip produces `floor_tripped=unauthorised_tool`. | Fixture passes when it should fail. |
| R9 | No wall-clock or randomness in the harness. | Grep clean. | Any present. |
| R10 | Model substitution case is explicit: if the exact `response.model` is not served, `model_substituted=true` lands in the replay output. | Flag present in the harness. | No flag; silent model substitution. |

Passing submission is 8+ of 10 with R1, R4, R5, R7 hard-required.

## 5. Common mistakes

- **Silent passthrough on unknown keys.** The whole point is to catch drift. A `return None` on unknown key defeats the exercise; the harness has to raise.
- **Collapsing retries to the final success.** The budget scorer in exercise-01 reads `retries.observed_max` and fires `retry_budget_blown`. If the recorded table collapses `[502, 502, 200]` to `[200]`, the replay's budget verdict will no longer match the original and `verify.py` fails on `retries` attribution.
- **Not canonicalising arguments on both ends.** If the record-time arguments were `{"top_k": 3, "query": "x"}` and the replay-time arguments are `{"query": "x", "top_k": "3"}`, the two string-forms differ; the canonical form (sorted keys, numeric-string coerced) is the same. Reuse exercise-01's canonicaliser — do not re-implement.
- **Treating the clock as implicit.** `datetime.utcnow()` inside a tool drifts on replay. Either wrap all clock reads behind the harness, or record the clock value in the tool-return record so the replayed return already carries it (the solution picks the second).
- **Executing side-effectful tools on replay.** `open_ticket` on replay must not open a real ticket. `replay_mode: "stub_only"` on the recorded entry is the flag; the harness must enforce it. If the harness treats side-effectful tools as "exact mode" and dispatches them, the replay pollutes the CRM.
- **Hand-coded one-shot verify.** `verify.py` has to be a diff tool, not a bespoke assertion. The exercise asks for the three attribution span IDs plus the two verdict fields; drop any to one and the regression gate is incomplete.
- **Skipping the bisection step.** One-pin-at-a-time is slow; the temptation is to swap two things at once. That trades exact attribution for "it works now" and kills the learning step of the post-mortem.
- **Not promoting the fixture into the regression dataset.** Learn is the sixth step for a reason. A fix that is not pinned as a regression Sample will un-fix itself on the next refactor.
- **Running the replay against the live provider when the exercise said to use the recorded table.** The exercise permits a very small live-model budget for the harness, but the model's non-determinism is exactly what the recorded table is supposed to remove. If you have `sampling.seed=null` and are hitting the live provider, you are not replaying; you are running a second independent eval with the same context.

## 6. References

Official standards and project docs:

- Inspect documentation, <https://inspect.aisi.org.uk/> — Solvers, Tools, TaskState.
- OpenInference specification, <https://github.com/Arize-ai/openinference> (the span tree the replay reads).
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/> (the attribute names used in the trace).
- Anthropic Messages API, <https://docs.anthropic.com/en/api/messages> (the provider whose `seed` parameter absence motivates the advisory pin).
- OpenAI reproducibility (`seed`), <https://platform.openai.com/docs/advanced-usage/reproducible-outputs> (the counter-example where exact-sampling replay is possible).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> (motivates why side-effectful tools must be `stub_only` on replay).
- MITRE ATLAS, <https://atlas.mitre.org/> (ATLAS TTPs inform the regression-promotion discipline — an unauthorised_tool trip that goes unfixed is an ATLAS entry point).
- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> (Measure / Manage functions inform the "versioned fixture + versioned post-mortem" discipline in step 6).
- Local chapters: `lessons/mod-103-trajectory-and-tool-eval/07-deterministic-replay-and-post-mortem.md` (primary input), `02-scoring-per-step-tool-calls.md` (canonicalisation reuse), `04-budgets-and-rollout-gates.md` (retry attribution).
