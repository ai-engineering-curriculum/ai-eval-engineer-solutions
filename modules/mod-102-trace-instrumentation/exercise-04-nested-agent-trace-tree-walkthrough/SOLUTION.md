# exercise-04 — Nested-Agent Trace-Tree Walkthrough — Solution

> Reference solution for `lessons/mod-102-trace-instrumentation/exercises/exercise-04-nested-agent-trace-tree-walkthrough.md`.
> Chapters 04 and 05 are the primary input. All six bisection steps are run against three seeded failures on a nested
> support-agent trace; artefacts are written in the shape mod-106 / mod-109 downstream code will consume.

## 1. Solution overview

This exercise tests whether the schema from exercise-03 and the backend from exercise-02 actually support chapter 05's six-step bisection on a real trace. The deliverable is not a single bug-fix write-up — it is **three bisection reports** that quote attribute values from the schema, three reproducer fixtures, three trace annotations on the backend, three review-queue tickets, and a `missed-signal.md` that feeds a schema patch back to exercise-03.

Five design choices anchor the reference solution:

1. **Extend exercise-01's Q&A service to a nested planner**, not a new agent. The suggested shape in the exercise — `LLM.plan → TOOL.lookup_order → TOOL.create_followup_email → LLM.reply`, with a GUARDRAIL post-reply — is the one the reference walks. It matches the chapter-04 schema one-to-one, so a reviewer can map attribute quotes back to the MUST tables without a translation layer.
2. **Seed three failures at *different* steps of the tree** so each report exercises a different signature from chapter 05's table: wrong tool arguments (plan-LLM span), silent tool failure (second TOOL span), over-refusal (GUARDRAIL span). This also keeps `missed-signal.md` non-empty — different failure classes expose different gaps.
3. **Report structure matches chapter 05 verbatim** — six headings in order, attribute quotes verbatim, no narration. This is the template mod-109 reviewers read and mod-106 fixture authors consume.
4. **Fixture JSON matches chapter 05's shape** and sits in a sibling `fixtures/` directory. A two-line harness (`python replay.py fixtures/run-1.json`) reproduces. If it doesn't reproduce, that is a data point, not a failure — chapter 05 says so.
5. **Annotations stay short.** One paragraph on the backend, machine-readable tag, numeric score. The longer story lives in the review-ticket file.

## 2. Worked implementation

### 2.1 The nested agent

Extended from exercise-01:

```python
# support_qa_nested.py
# AGENT.support_qa → LLM.plan → TOOL.lookup_order → TOOL.create_followup_email → LLM.reply → GUARDRAIL.output
# Instrumented with OpenInference primary + GenAI identity mirrors per the exercise-03 schema.

def answer(question: str) -> str:
    with tracer.start_as_current_span("agent.support_qa") as root:
        root.set_attributes({
            "openinference.span.kind": "AGENT",
            "app.surface": "support_qa",
            "app.step": "agent.support_qa",
            "session.id": new_session_id(),
            "user.id": user_hash(current_user()),
            "service.version": SERVICE_VERSION,
            "schema.version": SCHEMA_VERSION,
            "experiment.arm": CURRENT_ARM,
            "input.value": question,
        })
        plan = _llm_plan(question)
        if plan.tool_calls:
            lookup = _tool_lookup_order(plan.tool_calls[0])
            followup = _tool_create_followup(lookup)
            reply = _llm_reply(question, lookup, followup)
        else:
            reply = _llm_direct_reply(question)
        guarded = _guardrail_output(reply)
        root.set_attribute("output.value", _summarise(guarded))
        return guarded
```

Each `_*` function wraps a child span with the per-kind MUST attributes from exercise-03. The seeds below attack three different `_*` functions.

### 2.2 Seeded failures (the answer key)

| # | Seed                                                                                      | Expected root cause                                                | Chapter-05 signature       |
|---|-------------------------------------------------------------------------------------------|---------------------------------------------------------------------|----------------------------|
| 1 | Spiked plan-LLM system prompt instructs it to swap adjacent digits in any numeric id.     | `llm.plan.llm.tool_calls.0.tool_call.function.arguments.order_id` disagrees with `agent.support_qa.input.value`. | Wrong tool arguments |
| 2 | Stub `create_followup_email` returns HTTP 500 with `{"error":"smtp_down"}`.               | `tool.create_followup_email.status = ERROR`; `llm.reply` does not mention the failure.                          | Silent tool failure   |
| 3 | Naive guardrail fires on the word `cancel`; benign input `"please cancel my subscription"`. | `guardrail.output.app.guardrail_verdict = "block"` on benign input.                                              | Over-refusal          |

### 2.3 `bisection-run-1.md` — wrong tool arguments

```markdown
# bisection-run-1.md

Trace id: `9c8f42e0b5a7d1c3e24a0a…`
Backend: Phoenix (self-host). Cross-check attempted in Langfuse (notes in §Step 6).

## Step 1 — Right run?

Root-span attributes (quoted verbatim):

- `app.surface                = "support_qa"`
- `service.version            = "2026.11.05-abc1"`
- `experiment.arm             = "reply-model-v3"`
- `session.id                 = "sess_2e4a9f0a"`
- `user.id                    = "u_f0a1…"`
- `input.value                = "Where is my order 12345?"`
- `output.value               = "Followup email sent for order 12354."`

This is the right run — the complaint ("asked about 12345, got a followup for 12354") matches the root `input.value` and `output.value`.

## Step 2 — Failure signature

Wrong tool arguments (chapter 05's canonical signature). The tool was called on `order_id=12354`; the user asked about `12345`. Digit transposition.

## Step 3 — Top-down walk

Observed child-kind sequence:

```
AGENT.support_qa
 ├─ LLM.plan             (expected)
 ├─ TOOL.lookup_order    (expected)
 ├─ TOOL.create_followup_email (expected)
 ├─ LLM.reply            (expected)
 └─ GUARDRAIL.output     (expected)
```

No missing kind. No extra kind. No repeat. No wrong-order kind. No truncation.

## Step 4 — Descend to suspect span

The order id first appears on `LLM.plan`'s output:

- `LLM.plan.llm.input_messages.1.message.role    = "user"`
- `LLM.plan.llm.input_messages.1.message.content = "Where is my order 12345?"`
- `LLM.plan.llm.tool_calls.0.tool_call.function.name      = "lookup_order"`
- `LLM.plan.llm.tool_calls.0.tool_call.function.arguments = "{\"order_id\":\"12354\",\"reason\":\"no_shipping_update\"}"`

The user input carries `12345`; the plan LLM emitted `12354`. The smallest span that could have caused the failure is `LLM.plan` — the subsequent TOOL spans faithfully executed the wrong id.

Cross-check downstream to confirm no further corruption:

- `TOOL.lookup_order.input.value = "{\"order_id\":\"12354\"}"`
- `TOOL.lookup_order.output.value = "{\"status\":\"shipped\",\"eta\":\"…\"}"` (the 12354 order exists in the stub; nothing errored)
- `TOOL.create_followup_email.input.value = "{\"order_id\":\"12354\"}"`

Causality check passes: plan.output → lookup.input → followup.input carried `12354` consistently. The corruption happened at plan, not downstream.

## Step 5 — Bisect by attribute

Filter: last 200 traces where `experiment.arm = "reply-model-v3"` AND `app.surface = "support_qa"`, grouped by numeric-id parity between `agent.support_qa.input.value` and `tool.lookup_order.input.value`.

- Arm `reply-model-v3` (41 traces): **3** with numeric-id mismatch, all plan-LLM originating.
- Arm `reply-model-v2` (159 traces): **0** with numeric-id mismatch.

Hypothesis: the failure class is arm-correlated to `reply-model-v3`. Not a p-value; a hypothesis (chapter 05 note).

## Step 6 — Reproduce offline

Fixture — `fixtures/run-1.json`:

```json
{
  "surface": "support_qa",
  "experiment_arm": "reply-model-v3",
  "input": "Where is my order 12345?",
  "prompt_template_hash": "sha256:0f7e89a1…",
  "invocation_parameters": {"temperature": 0.2, "max_tokens": 256},
  "expected_tool_call": {
    "name": "lookup_order",
    "arguments": {"order_id": "12345", "reason": "no_shipping_update"}
  }
}
```

Harness: `python replay.py fixtures/run-1.json`. Reproduced in 3 of 5 replays (model-side non-determinism at `temperature=0.2`). The report flags that a `plan-only` unit assert — comparing `llm.tool_calls.0.tool_call.function.arguments.order_id` against the input's extracted id — is sufficient to catch the class. No E2E fixture needed.

Langfuse cross-check: same root cause landed in 61 s (vs 42 s in Phoenix). Difference is viewer rendering, not information available (`missed-signal.md` §run-1).
```

### 2.4 `bisection-run-2.md` — silent tool failure

```markdown
# bisection-run-2.md

Trace id: `3d7f02c1a9…`
Backend: Phoenix (self-host).

## Step 1 — Right run?

- `app.surface                = "support_qa"`
- `service.version            = "2026.11.05-abc1"`
- `experiment.arm             = "reply-model-v3"`
- `input.value                = "Where is my order 42?"`
- `output.value               = "I've confirmed your order status."`

Right run — the complaint ("I never got the followup email") matches an `output.value` that *promises* a status update but does not mention the email failure.

## Step 2 — Failure signature

Silent tool failure (chapter 05).

## Step 3 — Top-down walk

```
AGENT.support_qa
 ├─ LLM.plan
 ├─ TOOL.lookup_order
 ├─ TOOL.create_followup_email  ← status=ERROR
 ├─ LLM.reply
 └─ GUARDRAIL.output
```

All kinds present; no missing / extra / repeat / wrong-order. **Structural anomaly**: the second TOOL child carries `status = Error` but the trace did not terminate. The error did not surface in the reply.

## Step 4 — Descend to suspect span

- `TOOL.create_followup_email.status = ERROR`
- `TOOL.create_followup_email.output.value = "{\"error\":\"smtp_down\",\"retryable\":false}"`
- `LLM.reply.llm.input_messages.3.message.role = "tool"` (tool result threaded as a message)
- `LLM.reply.llm.input_messages.3.message.content = "{\"error\":\"smtp_down\",\"retryable\":false}"`
- `LLM.reply.llm.output_messages.0.message.content = "I've confirmed your order status."`

The reply LLM *saw* the tool error (it is on the `input_messages.3`) and chose not to surface it. The smallest span that could have caused the user-visible failure is `LLM.reply` — the plan-LLM and TOOL spans are correct about the ground truth; the reply model smoothed over it.

## Step 5 — Bisect by attribute

Filter: last 200 traces where `tool.create_followup_email.status = ERROR`, grouped by whether the reply mentions the failure (substring `email` or `followup` in `llm.reply.llm.output_messages.0.message.content`).

- Arm `reply-model-v3` (9 ERROR traces): **8** of 9 reply does not mention the failure.
- Arm `reply-model-v2` (11 ERROR traces): **2** of 11 do not mention.

The over-smoothing is arm-correlated. Attribution to arm — update the mod-107 scorer to penalise this.

## Step 6 — Reproduce offline

Fixture — `fixtures/run-2.json`:

```json
{
  "surface": "support_qa",
  "experiment_arm": "reply-model-v3",
  "input": "Where is my order 42?",
  "injected_tool_results": {
    "create_followup_email": {"error": "smtp_down", "retryable": false}
  },
  "expected_reply_contains": ["email", "could not"]
}
```

Harness: `python replay.py fixtures/run-2.json`. Reproduced in 5 of 5 replays. The plan-only unit test is insufficient; the fixture exercises the reply LLM with injected tool errors.

Missing from the schema: a stable field on the LLM reply span to indicate "the preceding tool failed." We can compute it post-hoc from `TOOL.*.status = Error` sibling check — but an `app.upstream_tool_error = true` on the reply span would make this filter one hop instead of two. Logged in `missed-signal.md`.
```

### 2.5 `bisection-run-3.md` — over-refusal

```markdown
# bisection-run-3.md

Trace id: `aa58f3…`
Backend: Phoenix (self-host).

## Step 1 — Right run?

- `app.surface                = "support_qa"`
- `service.version            = "2026.11.05-abc1"`
- `experiment.arm             = "reply-model-v3"`
- `input.value                = "I'd like to cancel my subscription please."`
- `output.value               = "I'm not able to help with that request."`

Right run — complaint matches ("refused a benign cancel-subscription question").

## Step 2 — Failure signature

Over-refusal (chapter 05).

## Step 3 — Top-down walk

```
AGENT.support_qa
 ├─ LLM.plan
 ├─ TOOL.lookup_order     ← skipped? no — present but trivial
 ├─ (no create_followup_email — not needed)
 ├─ LLM.reply
 └─ GUARDRAIL.output      ← verdict=block
```

Structural anomaly: the GUARDRAIL fired **after** a benign input.

## Step 4 — Descend to suspect span

- `GUARDRAIL.output.input.value  = "You can cancel your subscription on the Account page..."`
- `GUARDRAIL.output.output.value = "I'm not able to help with that request."`
- `GUARDRAIL.output.app.guardrail_verdict = "block"`
- `GUARDRAIL.output.app.guardrail_rule_hit = "contains_word:cancel"`
- `GUARDRAIL.output.app.guardrail_id = "policy_v14"`
- `GUARDRAIL.output.app.guardrail_version = "14.2"`

The naive rule `contains_word:cancel` fired on a legitimate cancellation workflow. Root cause is the rule, not the model. Smallest span: `GUARDRAIL.output`.

## Step 5 — Bisect by attribute

Filter: last 200 traces where `app.guardrail_verdict = "block"`, grouped by `app.guardrail_rule_hit`.

- `contains_word:cancel` (8 of 15 blocks): all benign cancellation requests. False-positive rate on this rule: 8/8.
- Other rule hits (7 of 15): mixed; not a clean signal.

Rule-level attribution: `contains_word:cancel` is the problem. Not model- or arm-correlated.

## Step 6 — Reproduce offline

Fixture — `fixtures/run-3.json`:

```json
{
  "surface": "support_qa",
  "input": "I'd like to cancel my subscription please.",
  "expected_guardrail_verdict": "allow",
  "guardrail_ruleset": "policy_v14@14.2"
}
```

Harness: `python replay.py fixtures/run-3.json`. Reproduced in 5 of 5 replays. Fix lies in the ruleset, not the model; mod-108 safety-eval work owns this but the fixture pins the regression.
```

### 2.6 `annotations.md`

```markdown
# Backend annotations

## Run 1 — trace 9c8f42e0b5a7d1c3e24a0a…

- URL: `http://localhost:6006/traces/9c8f42e0b5a7d1c3e24a0a`
- Label: `bug/tool-argument-format`
- Score: `app.judge_score = 1` (1–5 rubric; 1 = critical)
- Annotation (quoted verbatim):
  > Root cause: plan LLM emitted a digit-transposed tool-call `order_id` (12354 for 12345). Retriever / tool spans faithful. Reproducer at `fixtures/run-1.json`. Arm-correlated to `reply-model-v3`.

## Run 2 — trace 3d7f02c1a9…

- URL: `http://localhost:6006/traces/3d7f02c1a9`
- Label: `bug/silent-tool-failure`
- Score: `app.judge_score = 2`
- Annotation:
  > `tool.create_followup_email.status = Error` but `llm.reply` smoothed over it. Reproducer at `fixtures/run-2.json`. Schema gap flagged — see `missed-signal.md`.

## Run 3 — trace aa58f3…

- URL: `http://localhost:6006/traces/aa58f3`
- Label: `bug/over-refusal`
- Score: `app.judge_score = 2`
- Annotation:
  > GUARDRAIL fired on `contains_word:cancel` against a benign cancellation request. Rule-level false-positive; `policy_v14@14.2`. Reproducer at `fixtures/run-3.json`.
```

### 2.7 `review-tickets.md`

```markdown
# review-tickets.md

## Ticket — Run 1

- **Title**: `support_qa plan LLM emits digit-transposed tool-call order_id (arm reply-model-v3)`
- **Trace**: `http://localhost:6006/traces/9c8f42e0b5a7d1c3e24a0a`
- **Hypothesis**: `reply-model-v3` has a measurable rate of numeric-id digit transposition in tool-call arguments.
- **Proposed test**: plan-only assertion on `llm.tool_calls.0.tool_call.function.arguments.order_id == <extracted id from input>`; add as a mod-106 CI gate across a 50-trace sample.
- **Owner**: `@ai-eval-team` (support surface).

## Ticket — Run 2

- **Title**: `support_qa reply LLM smooths over tool errors (silent failure pattern)`
- **Trace**: `http://localhost:6006/traces/3d7f02c1a9`
- **Hypothesis**: `reply-model-v3` systematically does not surface `TOOL.status = Error` to the user.
- **Proposed test**: mod-107 online scorer that flags any trace where a sibling TOOL span has `status = Error` AND `llm.reply.output_messages.0.message.content` contains none of the words `["email","failed","could not","unable"]`.
- **Owner**: `@ai-eval-team` + `@onboarding-ops`.

## Ticket — Run 3

- **Title**: `support_qa guardrail policy_v14 fires on benign 'cancel' requests`
- **Trace**: `http://localhost:6006/traces/aa58f3`
- **Hypothesis**: the `contains_word:cancel` rule in `policy_v14@14.2` has a 100% false-positive rate on cancellation workflows.
- **Proposed test**: deprecate the rule or narrow it to multi-word patterns (`"cancel account"`, `"terminate subscription"`); verify against the `policy-v14-regression` fixture set (ownership shared with mod-108).
- **Owner**: `@safety-policy`.
```

### 2.8 `missed-signal.md`

```markdown
# missed-signal.md

Per-run schema gaps discovered during this exercise. Each feeds a `schema.version`
MINOR bump against exercise-03.

## Run 1 — wrong tool arguments

No gap — the schema carried everything needed. Observation on viewer speed: Phoenix rendered the plan-LLM tool-call JSON inline; Langfuse required one extra click to the Messages panel. This is a backend-rendering finding, not a schema finding.

## Run 2 — silent tool failure

**Gap**: no attribute on the LLM reply span indicates that a preceding TOOL span failed. Chapter-05 step 3 found the signal by scanning sibling TOOL spans; a single attribute would cut the walk to one hop.

**Proposed schema change** (additive, 1.0.0 → 1.1.0):

- On the `LLM` reply span: `app.upstream_tool_error` — `boolean`, MUST when a sibling `TOOL` span in the same AGENT has `status = Error`, computed and written by the instrumentation wrapper, not the model.
- On the `LLM` reply span: `app.upstream_tool_error.span_ids` — `string[]`, listing the failing TOOL span ids for cross-reference.

## Run 3 — over-refusal

**Gap**: `app.guardrail_rule_hit` was present (as a SHOULD in exercise-03's schema) but the exercise exposed that false-positive rates are impossible to roll up without it on *every* `block` or `rewrite` verdict.

**Proposed schema change** (strengthening — remains MINOR, 1.1.0 → 1.2.0):

- Promote `app.guardrail_rule_hit` from SHOULD to MUST on `GUARDRAIL` spans whose `app.guardrail_verdict` is `block` or `rewrite`. Allow absent on `allow`.
```

## 3. Validation steps

1. **Three bisection reports exist** with all six steps in order; each step quotes an attribute name and value (not narrative).
2. **Three fixture JSONs exist** under `fixtures/` and run via `python replay.py fixtures/run-<n>.json`. The replay harness is a two-line invocation.
3. **Three annotations are actually attached** on the backend — opening each trace URL shows the annotation, label, and score.
4. **Three review tickets** exist, each with a resolving trace URL and the four required fields (title, trace, hypothesis, proposed test, owner).
5. **`missed-signal.md`** is non-empty — at minimum run-2 or run-3 should surface a schema gap.
6. **Reproducer verdict stated** for every fixture — `reproduced` or `did not reproduce` plus a reason. "Did not reproduce" is a valid outcome; chapter 05 treats it as diagnostic.
7. **Bisection by attribute** is run — even a two-trace comparison counts; step 5 is the one people skip.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Three bisection-run-<n>.md files present. | All three. | Fewer than three. |
| R2 | Each report follows the chapter-05 six-step structure in order. | All six headings present in each file. | Steps missing or reordered. |
| R3 | Each report quotes attribute names *and* values. | `attribute = "value"` style per step. | "The model got confused." |
| R4 | Each report names the smallest span that could have caused the failure. | Named in step 4. | Named a whole subtree. |
| R5 | Three fixtures under `fixtures/` in the chapter-05 shape. | All three. | Shape differs or missing. |
| R6 | Fixtures run via a two-line harness; report states reproducer verdict. | `reproduced` / `did not reproduce` present. | No verdict. |
| R7 | Three annotations attached on the backend with label and score. | Visible when opening each URL. | Missing or unreachable. |
| R8 | Three review tickets with title / trace URL / hypothesis / test / owner. | All four fields filled. | Any field missing. |
| R9 | `missed-signal.md` names at least one schema gap. | Entry present. | Empty or "none." |
| R10 | Step 5 is non-trivial in every report — at least a two-trace comparison. | Present in all three. | Skipped. |

A passing submission is 8+ of 10 with R2, R3, R5, R6 all hard-required.

## 5. Common mistakes

- **Narrating the failure instead of quoting the span.** Chapter 05's anti-pattern. The fix is pasting the attribute name = value verbatim.
- **Skipping step 5.** The commonest skip and the one chapter 05 warns about. Even a two-trace comparison counts; "all examples use arm X, no comparison trace" is a legitimate finding if you say so.
- **Seeding all three failures at once, then bisecting them in parallel.** The exercise says do the seeds one at a time. Doing them in parallel causes cross-contamination — you will see a `block` from run 3 while debugging run 2.
- **Fixture shape drift.** The fixture is the mod-106 consumption format. Deviating from the chapter-05 shape (`surface`, `input`, `prompt_template_hash`, `invocation_parameters`, expected block) breaks downstream.
- **Reproducer verdict "reproduced" with no replay evidence.** Chapter 05: "If the fixture did not reproduce, the trace did not tell you the whole story." That is a *signal*, not a failure; name it.
- **Annotations as a wall of text.** The viewer annotation is one paragraph; the longer story is the review ticket.
- **`missed-signal.md` empty.** Per the exercise: "Empty is unusual; look harder." Three different failure classes against one schema almost always surface at least one gap.
- **Fixing at the wrong scope** in the proposed-test field. A plan-only bug does not need an E2E test; a reply-level over-smoothing bug does not need a retriever fix.
- **Not cross-checking on the second backend.** The stretch goal is cheap — note in `missed-signal.md` whether the other viewer would have led you to the same span faster. This is the data the exercise-02 memo also wants.
- **Writing step 3's "walk" as a paragraph.** It is a kind sequence; the ASCII tree is enough.

## 6. References

Official standards and project documentation:

- OpenTelemetry trace API — span links (used for sub-traces across fan-out), <https://opentelemetry.io/docs/specs/otel/trace/api/#link>.
- W3C Trace Context — `traceparent` / `tracestate` propagation (used in the cross-service stretch goal), <https://www.w3.org/TR/trace-context/>.
- OpenInference specification, <https://github.com/Arize-ai/openinference> (attribute names quoted in the bisection reports).
- OpenTelemetry semantic conventions — GenAI (mirror identity attributes), <https://opentelemetry.io/docs/specs/semconv/gen-ai/>.
- Arize Phoenix docs (backend annotations / scores), <https://docs.arize.com/phoenix>.
- Langfuse docs (backend annotations / scores for cross-check), <https://langfuse.com/docs>.

Local module chapters:

- `lessons/mod-102-trace-instrumentation/05-bisecting-a-bad-run.md` — the six-step bisection method, the common-failure-signature table, the "annotate and hand off" workflow.
- `lessons/mod-102-trace-instrumentation/04-span-schema-design.md` — the schema that the attribute quotes resolve against.
- `lessons/mod-102-trace-instrumentation/03-backend-bakeoff.md` — backend-level score / annotation hooks.
