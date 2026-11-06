# exercise-02 — Phoenix / Langfuse / Weave / Braintrust / LangSmith Bake-Off — Solution

> Reference solution for `lessons/mod-102-trace-instrumentation/exercises/exercise-02-phoenix-langfuse-weave-bakeoff.md`.
> Chapter 03 is the primary input; endpoint URLs, auth mechanics, and self-host feasibility change frequently and must
> be re-read from each vendor's current docs before the submission is accepted.

## 1. Solution overview

The exercise's job is not to pick a universal winner — it is to produce a *defensible* choice for *your* workload by running a scripted drill through a working OTel Collector that fans out to two deliberately-chosen backends. The reference solution walks the chapter 03 pattern end-to-end against one representative pairing (**Phoenix self-host + Langfuse SaaS**, a self-host-OpenInference-native vs SaaS-sessions-native comparison) and lays down a rubric that generalises to any of the recommended pairings.

Four design choices are load-bearing:

1. **Instrument once, fan out via a Collector.** The chapter 03 §"portable OTel-Collector recipe" is the exact shape; it keeps the service agnostic and makes the bake-off comparable (both backends see identical spans).
2. **Scrubbing and sampling come later (exercises 05).** The Collector here has one `batch` processor and no tail sampling or PII redaction; that is intentional, because this exercise measures viewer legibility and drill speed, not policy.
3. **Score on *wall-clock drill time*, not feature matrices.** The two least-fungible columns are "time to open a failing trace from a session id" and "time to identify the root-cause span on a seeded failure." A matrix of 10 scored features can be lied to; a stopwatch cannot.
4. **Never invent SaaS pricing, latency numbers, or self-host availability.** Chapter 03 is explicit: "do NOT hard-code latency, cost, ingest throughput." Everything cited in the memo links to a vendor doc URL read on the day of the bake-off.

## 2. Worked implementation

### 2.1 Chosen pairing and rationale

Reference pairing: **Phoenix (self-host) + Langfuse (SaaS)**.

- **Phoenix** is OpenInference-native and self-hostable. Chapter 03 names it as the low-friction default when the chapter 05 debugger workflow is primary.
- **Langfuse** has first-class session and user modelling in the UI; chapter 03 calls this out for chat / assistant surfaces where the *conversation* is the debugging unit. SaaS fast-path is a deliberate contrast to Phoenix's self-host story.

This pairing satisfies the "do not pick two vendors from the same camp" rule — different convention leanings, different hosting defaults, different primary artefacts. Equivalent reference pairings (Langfuse self-host + Braintrust SaaS; Weave SaaS + Phoenix self-host) use the same scoring template; the rubric does not change.

### 2.2 `collector.yaml` — working fan-out

```yaml
# collector.yaml — fans out traces from exercise-01 to Phoenix and Langfuse.
# Per chapter 03: read each vendor's OpenTelemetry-ingest page before copy-pasting.
#   Phoenix:  https://docs.arize.com/phoenix
#   Langfuse: https://langfuse.com/docs — search "OpenTelemetry" for the current
#             OTLP endpoint URL, basic-auth header shape, and project-id header.
# Processor pipeline is deliberately minimal here. tail_sampling and attribute
# redaction come in exercise-05.

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  batch:
    timeout: 1s
    send_batch_size: 512

exporters:
  otlphttp/phoenix:
    # Self-hosted Phoenix container running on port 6006 (docker image
    # arizephoenix/phoenix:latest). Confirm the current tag on
    # https://docs.arize.com/phoenix before pinning.
    endpoint: "http://localhost:6006"
    headers: {}

  otlphttp/langfuse:
    # Langfuse OTLP ingest URL. The path is documented on the vendor's
    # OpenTelemetry docs page — the shape below reflects the region-prefixed
    # form in Langfuse's current docs; re-read before deploy.
    endpoint: "${LANGFUSE_OTLP_ENDPOINT}"
    headers:
      # Basic auth = base64(public_key:secret_key) per Langfuse docs.
      authorization: "Basic ${LANGFUSE_BASIC_AUTH}"

service:
  telemetry:
    logs:
      level: info
  pipelines:
    traces:
      receivers: [otlp]
      processors: [batch]
      exporters: [otlphttp/phoenix, otlphttp/langfuse]
```

Deployment:

```bash
# Phoenix container (viewer + OTLP receiver on 6006 for HTTP, 4317 for gRPC).
docker run -d --name phoenix -p 6006:6006 -p 4317:4317 arizephoenix/phoenix:latest

# Collector — bind to the host so the Python service on the host can reach it
# at http://localhost:4318. On Linux, the collector talks to Phoenix on
# host.docker.internal:6006 or the container network — adjust per the host.
docker run -d --name otelcol \
  -v "$(pwd)/collector.yaml:/etc/otelcol-contrib/config.yaml" \
  -p 4317:4317 -p 4318:4318 \
  -e LANGFUSE_OTLP_ENDPOINT -e LANGFUSE_BASIC_AUTH \
  otel/opentelemetry-collector-contrib:latest \
  --config=/etc/otelcol-contrib/config.yaml

# Validate (the step chapter 03 names explicitly).
docker exec otelcol otelcol-contrib validate --config=/etc/otelcol-contrib/config.yaml
```

Point the exercise-01 service at the Collector:

```bash
export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:4318
python app_openinference.py
```

Secrets stay in env (`LANGFUSE_BASIC_AUTH`, `LANGFUSE_OTLP_ENDPOINT`). The acceptance criterion is zero secrets in the committed YAML.

### 2.3 Drill traffic generator

```python
# drill_traffic.py — ~200 traces including ~5 seeded failures.
import random
from app_openinference import answer  # exercise-01 OpenInference variant

NORMAL_INPUTS = [
    "Where is order 42?",
    "What's the ETA on 87?",
    "Do you ship to Iceland?",
    "Hi there.",
    "Tell me a joke about printers.",
]

SEEDED = [
    # 1. bad tool argument — nonsense order id
    "Where is order !@#$?",
    # 2. empty input
    "",
    # 3. very long input — should produce a long response
    "Please list every possible reason a package might be delayed, in detail: " + ("A" * 3500),
    # 4. tripwire content — asks about a word any naive filter would hit
    "Please cancel my subscription and close my account.",
    # 5. nonexistent order id — tool returns an error payload
    "Where is order 999999?",
]


def main() -> None:
    bag = NORMAL_INPUTS * 40 + SEEDED
    random.shuffle(bag)
    assert len(bag) >= 200
    for i, q in enumerate(bag[:205]):
        try:
            _ = answer(q or "(empty)")
        except Exception as exc:  # noqa: BLE001
            print(f"[{i}] failed: {exc!r}")
        if i % 25 == 0:
            print(f"progress: {i}")


if __name__ == "__main__":
    main()
```

Budget guard: ~205 traces × ~2 model calls × `max_tokens=256` is well under $5 on `gpt-4o-mini` (the chapter 03 budget frame).

### 2.4 `bakeoff-scorecard.md` — the actual deliverable

```markdown
# Backend bake-off scorecard — support_qa surface

Traffic: ~205 traces generated via `drill_traffic.py` on 2026-11-05; both backends
received the same spans through the Collector. Wall-clock times are median over the
stated repetitions, taken by a single on-call (me). Peer-measurement adjustment is
noted where relevant.

| Column                                                 | Phoenix (self-host)                                    | Langfuse (SaaS)                                      |
|--------------------------------------------------------|--------------------------------------------------------|-------------------------------------------------------|
| Convention leaning (and viewer rendering)              | OpenInference-native. Indexed `llm.input_messages.<i>.message.content` renders as chat bubbles; tool-call widget shows `function.name` + JSON `arguments`; RETRIEVER renders per-document scores and content side-by-side. `gen_ai.*` mirrors render as raw key/value in the "Attributes" tab. | Vendor-native data model (traces / sessions / scores). OTLP ingest accepted; OpenInference-shaped spans render under a trace's spans list with a messages panel per LLM span. Session view groups spans by `session.id`. |
| Trace-tree legibility (1–5)                            | 5 — chapter 05's top-down walk takes one scan; the AGENT root's `input.value` / `output.value` are shown at the top of the trace detail. | 4 — the trace tree is legible; sessions add one layer above but costs one click to reach a single trace's root. |
| Session model                                          | Present — `session.id` renders as a filterable attribute but is not a top-level UI object. Query-first. | First-class — Sessions is a top-nav object; a chat's sessions appear as a list with trace counts. |
| Median wall-clock to open a specific failing trace from a session id (5 attempts) | 12 s — paste `session.id` into the trace-search bar, open the matching trace. | 7 s — sessions nav → session row → trace list. |
| Median wall-clock to identify root-cause span on a seeded failure (3 drills) | 48 s — OpenInference rendering lets you skim the plan LLM's tool-call JSON in the first bubble. | 65 s — requires one extra click to open the messages panel; messages panel renders content well once opened. |
| Ease of attaching a numeric score to a span (1–5) + example call | 5 — Phoenix exposes a score API; example: `px.Client().log_evaluations(SpanEvaluations(eval_name="judge", dataframe=df))` (see `https://docs.arize.com/phoenix` for the current SDK). | 5 — `langfuse_client.score(trace_id=..., name="judge", value=1)` or `observation.score(...)` (see `https://langfuse.com/docs`). |
| Self-host feasibility                                  | Yes — `arizephoenix/phoenix` container documented on `https://docs.arize.com/phoenix`; repo `https://github.com/Arize-ai/phoenix`. | Yes — self-host story documented on `https://langfuse.com/docs/deployment/self-host`. (Re-verify URL.) |
| PII / hosting posture                                  | Payload stays inside the VPC in the self-host image; no third-party egress unless you add one. | SaaS by default — payload leaves the VPC to the Langfuse region host. Self-host option exists (see cell above). Compliance frame determines fit. |
| Notable friction                                       | OpenInference rendering is rich but `gen_ai.*` mirrors sit in a separate tab; dual-shipped traces look "half-rendered" unless you pick primary clearly. | SaaS auth via base64(public_key:secret_key) header is easy to mis-paste; validate `collector.yaml` before pointing production at it. |
```

### 2.5 `drill-log.md` — raw drill records

Three seeded failures, one section per failure, one block per backend. Example block (truncated for the SOLUTION; a full submission runs the same shape across all three):

```markdown
## Drill 1 — Wrong tool argument (`Where is order !@#$?`)

### Phoenix
- Trace filter: `input.value` contains `"!@#$"`.
- Click path: Traces → filter bar → open single result → expand `llm.plan` → open "Messages" panel.
- Root cause attribute + value:
  - `llm.tool_calls.0.tool_call.function.arguments` = `{"order_id": "!@#$"}`
  - `tool.lookup_order.output.value` = `{"error":"unknown_order","order_id":"!@#$"}`
- Wall-clock: 42 s.

### Langfuse
- Trace filter: Observations search on `input.value` containing `!@#$`.
- Click path: Traces → filter → open trace → expand `llm.plan` → "I/O" panel.
- Root cause attribute + value:
  - `llm.tool_calls.0.tool_call.function.arguments` = `{"order_id": "!@#$"}`
  - `tool.lookup_order.output.value` = `{"error":"unknown_order","order_id":"!@#$"}`
- Wall-clock: 58 s.
```

Repeat for Drills 2 (long response) and 3 (nonexistent order id). Keep the attribute quotes verbatim; the point of the log is to let chapter 05's workflow be run from the backend without going back to the source code.

### 2.6 `memo.md` — pick-a-backend memo

```markdown
# Backend recommendation — support_qa surface

## Recommendation

Phoenix (self-host) as the primary debugging backend for `support_qa`. Keep Langfuse live for four weeks as a mirror to (a) confirm the session-first debugging thesis against real on-call usage and (b) give the compliance review a comparison before committing fully. If no on-call opens Langfuse in the four-week window, retire the mirror.

## Reasoning

The scorecard separates the two on two columns that matter: median time to root-cause and payload-residency posture. Phoenix's OpenInference-native rendering shaves ~15 s off the root-cause-span drill (48 s median vs 65 s on Langfuse) because the plan-LLM tool-call renders inline as a chat bubble rather than requiring an extra click into a "messages" panel. For a workflow chapter 05 says we should be able to run in twenty minutes, saving a click on the hottest step compounds. Phoenix also keeps payload inside the VPC with the published self-host image; Langfuse SaaS is a payload-leaves-the-VPC posture that costs a Governance-family sign-off we would otherwise not need.

Langfuse beat Phoenix on "time to open a trace from a session id" (7 s vs 12 s) because sessions are first-class in the nav; for a chat surface this is a real win. The decision tips to Phoenix because the root-cause drill is the surface's dominant on-call activity, and because the compliance posture is easier to defend. For a chat-heavy surface where root cause is rarer (a meeting summariser, perhaps), the same scorecard would flip to Langfuse.

## What the bake-off did *not* answer

- **Throughput at 10× load.** The drill ran 205 traces; at 10,000 / min we have no observation. The chapter 03 note "do NOT hard-code throughput numbers" applies. Escalate via a load test before a GA cutover.
- **Cost at production volume.** Vendor unit prices (storage per million spans, judge-score tier pricing) were not priced in because they change monthly (chapter 03). Procurement owns this before the cutover.
- **EU data residency on the SaaS option.** Langfuse's current EU region support is on their docs page; a single read on 2026-11-05 is not a substitute for a security review. Hand off to the Security peer.
- **LangChain / LangGraph fit.** The surface is homegrown; a future LangChain migration might flip the recommendation to LangSmith. Mark as "reassess on framework change."

## Migration cost

If we ship on Phoenix and want to flip to Langfuse in six months, the change is small: the Collector gains a Langfuse exporter and eventually loses the Phoenix one. The service does not change — it emits OTel OTLP either way. Dashboards (chapter 06 / mod-111) and judge-score attach logic (mod-107) are backend-specific and would need rewriting against Langfuse's score API; that is roughly one engineer-week.
```

## 3. Validation steps

1. **Collector validates**: `otelcol-contrib validate --config=collector.yaml` returns success.
2. **One live trace reaches both backends**: send one request, confirm the trace appears in both viewers within a minute.
3. **Full drill lands**: run `drill_traffic.py`, confirm ~205 traces are present in both viewers within a minute of the final line. Spot-check 3 traces in each viewer: inputs match, final outputs match, span counts match.
4. **Scorecard numbers are reproducible**: a second person performs the three drills on both backends; their wall-clock medians should sit within ~30% of the submitter's. Chapter 03 calls this out as the "2am on-call" test.
5. **Secrets check**: `git grep -E '(sk-|pk-|Basic [A-Za-z0-9+/=]{20,})'` returns nothing in the committed YAML.
6. **Vendor URLs resolve**: every URL cited in the memo and scorecard loads on the day of review.
7. **Memo passes the chapter-03 word check**: no "leading", "industry-standard", or "best-in-class" without a quoted vendor doc URL.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | `collector.yaml` validates cleanly. | `otelcol-contrib validate` exits 0. | Validation error. |
| R2 | Fan-out works: one trace visible in both viewers after one run. | Both viewers show the same trace id (allowing for vendor-side renaming). | Only one viewer renders. |
| R3 | ~200 traces in both viewers within a minute of the drill script finishing. | Trace count matches on both sides (±5 for ingest timing). | Significant drop on one side. |
| R4 | Scorecard has all ten columns filled for both backends, with wall-clock medians drawn from the drill log. | Every cell non-empty; wall-clock cells cite the median of 3–5 attempts. | "TBD" or marketing-copy cells. |
| R5 | `drill-log.md` quotes attribute names *and* values for every drill-backend pair. | `llm.tool_calls.0.tool_call.function.arguments = "{…}"` style evidence. | Narrative "the model got confused." |
| R6 | Memo names a primary, a reason, an admission, and a migration estimate. | All four sections present and non-empty. | Any section missing. |
| R7 | No vendor marketing language without a quoted docs URL. | Every superlative links to a vendor docs page. | "LangSmith is industry-leading" with no URL. |
| R8 | No secret in the committed Collector config or memo. | `git grep` for key patterns is clean. | Any secret present. |
| R9 | Vendor self-host / hosting posture cells cite the vendor's current docs URL. | Each cell has a URL. | Claim without URL. |
| R10 | Second-person drill times exist for at least one backend (R4 adjustment or stretch goal). | Peer wall-clock medians included for one backend. | Single-person data only. |

A passing submission is 8+ of 10 with R1, R2, R6, R8 all hard-required.

## 5. Common mistakes

- **Copy-pasting the chapter-03 YAML verbatim.** Chapter 03 names this explicitly: "do not copy the chapter-03 sketch verbatim." Endpoints and auth header shapes have moved in past releases; re-read the vendor docs.
- **Forgetting basic-auth encoding for Langfuse.** The ingest header is `Basic base64(public_key:secret_key)`. Pasting the keys directly produces 401s that the Collector reports but the service does not see.
- **Measuring drill times from memory.** Wall-clock is the least-fungible column; use an actual stopwatch per chapter 03.
- **"Trace-tree legibility: 5" with no justification.** The scorecard cell needs a one-sentence reason per chapter 03.
- **Adding `tail_sampling` or `redaction` processors now.** They belong to exercise-05. Doing them here conflates "legibility" with "cost control."
- **Running the drill through only one backend.** Fan-out means *both* viewers see every trace. If one is empty, the exporter is mis-configured.
- **Writing the memo from marketing pages.** R7 catches this; chapter 03 is explicit about the vocabulary rule.
- **Picking two SaaS vendors from the same camp.** Violates the "do not pick two from the same camp" rule; the bake-off loses its point because the second backend is a near-duplicate of the first.
- **Claiming a feature that is behind a paywall as if it is available.** Chapter 02 rule: mark features behind enterprise plans clearly and lower the score with a note; do NOT infer capabilities you did not exercise.
- **Not force-flushing the SDK at the end of the drill script.** The last ~50 traces will not reach the Collector.

## 6. References

Official standards and project docs cited:

- OpenTelemetry Collector, <https://opentelemetry.io/docs/collector/> (pipeline model, deployment).
- `opentelemetry-collector-contrib`, <https://github.com/open-telemetry/opentelemetry-collector-contrib> (receivers / processors / exporters).
- OpenTelemetry Protocol (OTLP), <https://opentelemetry.io/docs/specs/otlp/>.
- W3C Trace Context, <https://www.w3.org/TR/trace-context/> (cross-service propagation, for the stretch goal).

Vendor documentation (verify each URL on the day of the bake-off):

- Arize Phoenix, <https://docs.arize.com/phoenix>; repo <https://github.com/Arize-ai/phoenix>.
- Langfuse, <https://langfuse.com/docs>; self-host <https://langfuse.com/docs/deployment/self-host>.
- W&B Weave, <https://weave-docs.wandb.ai/>.
- Braintrust, <https://www.braintrust.dev/docs>.
- LangSmith, <https://docs.smith.langchain.com/>.

Local module chapters:

- `lessons/mod-102-trace-instrumentation/03-backend-bakeoff.md` — vendor comparison matrix, Collector recipe, decision guide.
- `lessons/mod-102-trace-instrumentation/05-bisecting-a-bad-run.md` — six-step bisection used by the root-cause drill.
