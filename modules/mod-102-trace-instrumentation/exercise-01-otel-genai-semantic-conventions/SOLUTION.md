# exercise-01 — OTel GenAI and OpenInference Semantic Conventions — Solution

> Reference solution for `lessons/mod-102-trace-instrumentation/exercises/exercise-01-otel-genai-semantic-conventions.md`.
> Chapters 01 and 02 of the learning module are the primary input; attribute names in this file are cited from the
> OpenTelemetry GenAI semantic conventions and the OpenInference spec (URLs in §6).

## 1. Solution overview

The exercise produces two side-by-side instrumentations of the *same* `answer(question)` service — once with **OpenTelemetry GenAI** attributes (plus content on span events) and once with **OpenInference** attributes (plus content on indexed sub-attributes) — and a 400–600-word `compare.md` that reads the two traces off a real viewer. The deliverable is the diff and the recommendation, not the code; the code is scaffolding for the diff.

Three deliberate design choices anchor the solution:

1. **One convention per span, consistently.** Chapter 02 is explicit: "pick one as **primary per span** and stay consistent." Mixing names on the same span makes rendering unreliable; the two files make the contrast visible.
2. **Content shape matches convention.** The OTel GenAI variant carries message content on **span events** (`gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.system.message`, `gen_ai.choice`) because that is the historical shape the chapter calls out. The OpenInference variant carries content on **indexed attributes** (`llm.input_messages.<i>.message.content`). The chapter notes the OTel GenAI spec is evolving toward attributes as well — the exercise freezes the historical shape so the diff is readable.
3. **Manual instrumentation only, no auto-instrumentor.** The point is to feel each attribute name; autoinstrumentors hide that.

The recommendation in `compare.md` is a **dual-annotation / primary-plus-mirror** stance (chapter 02's §"Example: dual-convention on the same LLM span"): primary OpenInference, mirror the identity attributes under `gen_ai.*`. The reasoning sits in the rendering diff — the OpenInference variant lights up as structured chat / tool-call widgets in Phoenix, which is the on-call's actual workflow (chapter 05), while the GenAI mirror preserves interop for backends that lean the other way. The dual pattern is the chapter 02 answer; dropping one convention entirely is a legitimate alternative when the backend posture is fixed.

## 2. Worked implementation

The two files below are the reference shape. They are intentionally close in structure so the *attribute names* are what differs; they diverge only where the convention requires it (events vs indexed attributes, `SpanKind` vs `openinference.span.kind`).

### 2.1 `app_otel_genai.py` — OpenTelemetry GenAI variant

```python
"""Support Q&A with one tool, instrumented with OpenTelemetry GenAI conventions.

Attribute and event names cited from:
  https://opentelemetry.io/docs/specs/semconv/gen-ai/
Status: the GenAI semantic conventions are in development / experimental;
re-read the URL before pinning names for production.
"""
from __future__ import annotations

import json
import os
from typing import Any

from openai import OpenAI
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.trace import SpanKind, Status, StatusCode

RESOURCE = Resource.create({SERVICE_NAME: "support_qa_otel_genai"})
PROVIDER = TracerProvider(resource=RESOURCE)
PROVIDER.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(
            endpoint=os.environ.get(
                "OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4318"
            )
            + "/v1/traces"
        )
    )
)
trace.set_tracer_provider(PROVIDER)
tracer = trace.get_tracer("support_qa.otel_genai")

MODEL = "gpt-4o-mini"  # pin to a cheap, deterministic-ish model for the exercise
SYSTEM_PROMPT = (
    "You are a concise support agent. Call lookup_order(order_id) if a specific "
    "order id is mentioned; otherwise answer directly."
)
ORDERS = {
    "42": {"status": "shipped", "eta": "2026-11-07"},
    "87": {"status": "processing", "eta": "2026-11-12"},
    # three more…
}
TOOL_SCHEMA = [
    {
        "type": "function",
        "function": {
            "name": "lookup_order",
            "description": "Look up an order by id",
            "parameters": {
                "type": "object",
                "properties": {"order_id": {"type": "string"}},
                "required": ["order_id"],
            },
        },
    }
]

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])


def lookup_order(order_id: str) -> dict[str, Any]:
    return ORDERS.get(order_id, {"error": "unknown_order", "order_id": order_id})


def _record_llm(span, operation: str, request_model: str) -> None:
    # Identity attributes — chapter 02 Table (gen_ai.* core set).
    span.set_attribute("gen_ai.system", "openai")
    span.set_attribute("gen_ai.operation.name", operation)  # "chat"
    span.set_attribute("gen_ai.request.model", request_model)
    span.set_attribute("gen_ai.request.temperature", 0.2)
    span.set_attribute("gen_ai.request.max_tokens", 256)


def _record_response(span, resp) -> None:
    span.set_attribute("gen_ai.response.id", resp.id)
    span.set_attribute("gen_ai.response.model", resp.model)
    span.set_attribute("gen_ai.usage.input_tokens", resp.usage.prompt_tokens)
    span.set_attribute("gen_ai.usage.output_tokens", resp.usage.completion_tokens)
    span.set_attribute(
        "gen_ai.response.finish_reasons",
        [c.finish_reason or "" for c in resp.choices],
    )


def answer(question: str) -> str:
    with tracer.start_as_current_span(
        "support_qa.answer",
        kind=SpanKind.INTERNAL,
        attributes={"gen_ai.operation.name": "invoke_agent"},
    ):
        messages: list[dict[str, Any]] = [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": question},
        ]

        # First model call: may emit a tool call.
        with tracer.start_as_current_span("llm.plan", kind=SpanKind.CLIENT) as span:
            _record_llm(span, "chat", MODEL)
            # Content on events (historical GenAI shape, chapter 02).
            span.add_event(
                "gen_ai.system.message", attributes={"content": SYSTEM_PROMPT}
            )
            span.add_event("gen_ai.user.message", attributes={"content": question})

            resp = client.chat.completions.create(
                model=MODEL,
                messages=messages,
                tools=TOOL_SCHEMA,
                temperature=0.2,
                max_tokens=256,
            )
            _record_response(span, resp)
            choice = resp.choices[0]

            span.add_event(
                "gen_ai.choice",
                attributes={
                    "index": 0,
                    "finish_reason": choice.finish_reason or "",
                    "content": choice.message.content or "",
                },
            )

        tool_calls = choice.message.tool_calls or []
        if not tool_calls:
            return choice.message.content or ""

        # Execute the stubbed tool — INTERNAL span per chapter 02.
        call = tool_calls[0]
        with tracer.start_as_current_span(
            "tool.lookup_order",
            kind=SpanKind.INTERNAL,
            attributes={
                "gen_ai.operation.name": "execute_tool",
                "gen_ai.tool.name": call.function.name,
                "gen_ai.tool.call.id": call.id,
            },
        ) as tspan:
            args = json.loads(call.function.arguments or "{}")
            result = lookup_order(args["order_id"])
            tspan.set_attribute("gen_ai.tool.call.arguments", call.function.arguments)
            tspan.set_attribute("gen_ai.tool.call.result", json.dumps(result))
            if "error" in result:
                tspan.set_status(Status(StatusCode.ERROR, result["error"]))

        # Second model call with the tool result.
        messages.append(choice.message.model_dump(exclude_unset=True))
        messages.append(
            {
                "role": "tool",
                "tool_call_id": call.id,
                "content": json.dumps(result),
            }
        )
        with tracer.start_as_current_span("llm.reply", kind=SpanKind.CLIENT) as span:
            _record_llm(span, "chat", MODEL)
            for m in messages:
                event_name = {
                    "system": "gen_ai.system.message",
                    "user": "gen_ai.user.message",
                    "assistant": "gen_ai.assistant.message",
                    "tool": "gen_ai.tool.message",
                }[m["role"]]
                span.add_event(event_name, attributes={"content": str(m.get("content") or "")})

            resp = client.chat.completions.create(
                model=MODEL, messages=messages, temperature=0.2, max_tokens=256
            )
            _record_response(span, resp)
            reply = resp.choices[0].message.content or ""
            span.add_event(
                "gen_ai.choice",
                attributes={"index": 0, "content": reply, "finish_reason": "stop"},
            )
        return reply


INPUTS = [
    "Where is order 42?",
    "What's the ETA on 87?",
    "Do you ship to Iceland?",
    "Hi there.",
    "Tell me a joke about printers.",
]

if __name__ == "__main__":
    for q in INPUTS:
        print(f"Q: {q}\nA: {answer(q)}\n")
    PROVIDER.force_flush()
```

Why each attribute is where it is:

- **Identity set** (`gen_ai.system`, `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.*`) is the chapter 02 table of core GenAI attributes. Every LLM span carries it.
- **`SpanKind.CLIENT`** on the LLM call because the HTTP request leaves the process; **`SpanKind.INTERNAL`** on the tool execution because the stub runs in-process (chapter 02 / [OTel SpanKind spec](https://opentelemetry.io/docs/specs/otel/trace/api/#spankind)).
- **Content on events** (`gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.system.message`, `gen_ai.choice`, `gen_ai.tool.message`) is the historical shape; the chapter notes the spec is evolving toward attributes. The exercise freezes the event shape to make the diff legible.
- **Tool attributes on the tool span** (`gen_ai.tool.name`, `gen_ai.tool.call.id`) plus the LLM-side `gen_ai.operation.name=execute_tool` — chapter 02 lists these as the GenAI tool-call vocabulary.

### 2.2 `app_openinference.py` — OpenInference variant

```python
"""Support Q&A with one tool, instrumented with OpenInference conventions.

Attribute names cited from the OpenInference spec:
  https://github.com/Arize-ai/openinference/tree/main/spec
Primary classifier: openinference.span.kind. Content lives on indexed
sub-attributes (llm.input_messages.<i>.message.content), not events.
"""
from __future__ import annotations

import json
import os
from typing import Any

from openai import OpenAI
from opentelemetry import trace
from opentelemetry.exporter.otlp.proto.http.trace_exporter import OTLPSpanExporter
from opentelemetry.sdk.resources import SERVICE_NAME, Resource
from opentelemetry.sdk.trace import TracerProvider
from opentelemetry.sdk.trace.export import BatchSpanProcessor
from opentelemetry.trace import SpanKind

RESOURCE = Resource.create({SERVICE_NAME: "support_qa_openinference"})
PROVIDER = TracerProvider(resource=RESOURCE)
PROVIDER.add_span_processor(
    BatchSpanProcessor(
        OTLPSpanExporter(
            endpoint=os.environ.get(
                "OTEL_EXPORTER_OTLP_ENDPOINT", "http://localhost:4318"
            )
            + "/v1/traces"
        )
    )
)
trace.set_tracer_provider(PROVIDER)
tracer = trace.get_tracer("support_qa.openinference")

MODEL = "gpt-4o-mini"
SYSTEM_PROMPT = (
    "You are a concise support agent. Call lookup_order(order_id) if a specific "
    "order id is mentioned; otherwise answer directly."
)
ORDERS = {"42": {"status": "shipped", "eta": "2026-11-07"},
          "87": {"status": "processing", "eta": "2026-11-12"}}
TOOL_SCHEMA = [{
    "type": "function",
    "function": {
        "name": "lookup_order",
        "description": "Look up an order by id",
        "parameters": {"type": "object",
                       "properties": {"order_id": {"type": "string"}},
                       "required": ["order_id"]},
    },
}]

client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])


def lookup_order(order_id: str) -> dict[str, Any]:
    return ORDERS.get(order_id, {"error": "unknown_order", "order_id": order_id})


def _record_llm_request(span, messages: list[dict[str, Any]]) -> None:
    span.set_attribute("openinference.span.kind", "LLM")
    span.set_attribute("llm.provider", "openai")
    span.set_attribute("llm.model_name", MODEL)
    span.set_attribute(
        "llm.invocation_parameters",
        json.dumps({"temperature": 0.2, "max_tokens": 256}),
    )
    for i, m in enumerate(messages):
        span.set_attribute(f"llm.input_messages.{i}.message.role", m["role"])
        span.set_attribute(
            f"llm.input_messages.{i}.message.content", str(m.get("content") or "")
        )
    span.set_attribute("llm.tools", json.dumps(TOOL_SCHEMA))
    span.set_attribute("input.value", messages[-1]["content"])
    span.set_attribute("input.mime_type", "text/plain")


def _record_llm_response(span, resp) -> None:
    choice = resp.choices[0]
    span.set_attribute("llm.model_name", resp.model)  # observed served model
    span.set_attribute("llm.token_count.prompt", resp.usage.prompt_tokens)
    span.set_attribute("llm.token_count.completion", resp.usage.completion_tokens)
    span.set_attribute("llm.token_count.total", resp.usage.total_tokens)
    span.set_attribute(
        "llm.output_messages.0.message.role", "assistant"
    )
    span.set_attribute(
        "llm.output_messages.0.message.content", choice.message.content or ""
    )
    for i, tc in enumerate(choice.message.tool_calls or []):
        span.set_attribute(
            f"llm.tool_calls.{i}.tool_call.function.name", tc.function.name
        )
        span.set_attribute(
            f"llm.tool_calls.{i}.tool_call.function.arguments",
            tc.function.arguments or "",
        )
    span.set_attribute("output.value", choice.message.content or "")
    span.set_attribute("output.mime_type", "text/plain")


def answer(question: str) -> str:
    with tracer.start_as_current_span("support_qa.answer", kind=SpanKind.INTERNAL) as root:
        root.set_attribute("openinference.span.kind", "AGENT")
        root.set_attribute("input.value", question)

        messages: list[dict[str, Any]] = [
            {"role": "system", "content": SYSTEM_PROMPT},
            {"role": "user", "content": question},
        ]

        with tracer.start_as_current_span("llm.plan", kind=SpanKind.CLIENT) as span:
            _record_llm_request(span, messages)
            resp = client.chat.completions.create(
                model=MODEL,
                messages=messages,
                tools=TOOL_SCHEMA,
                temperature=0.2,
                max_tokens=256,
            )
            _record_llm_response(span, resp)
            choice = resp.choices[0]

        tool_calls = choice.message.tool_calls or []
        if not tool_calls:
            reply = choice.message.content or ""
            root.set_attribute("output.value", reply)
            return reply

        call = tool_calls[0]
        with tracer.start_as_current_span(
            f"tool.{call.function.name}", kind=SpanKind.INTERNAL
        ) as tspan:
            tspan.set_attribute("openinference.span.kind", "TOOL")
            tspan.set_attribute("tool.name", call.function.name)
            tspan.set_attribute("tool.description", "Look up an order by id")
            tspan.set_attribute("tool.parameters", json.dumps(TOOL_SCHEMA[0]["function"]["parameters"]))
            tspan.set_attribute("input.value", call.function.arguments or "")
            args = json.loads(call.function.arguments or "{}")
            result = lookup_order(args["order_id"])
            tspan.set_attribute("output.value", json.dumps(result))

        messages.append(choice.message.model_dump(exclude_unset=True))
        messages.append(
            {"role": "tool", "tool_call_id": call.id, "content": json.dumps(result)}
        )
        with tracer.start_as_current_span("llm.reply", kind=SpanKind.CLIENT) as span:
            _record_llm_request(span, messages)
            resp = client.chat.completions.create(
                model=MODEL, messages=messages, temperature=0.2, max_tokens=256
            )
            _record_llm_response(span, resp)
            reply = resp.choices[0].message.content or ""

        root.set_attribute("output.value", reply)
        return reply


INPUTS = [
    "Where is order 42?",
    "What's the ETA on 87?",
    "Do you ship to Iceland?",
    "Hi there.",
    "Tell me a joke about printers.",
]

if __name__ == "__main__":
    for q in INPUTS:
        print(f"Q: {q}\nA: {answer(q)}\n")
    PROVIDER.force_flush()
```

Why each attribute is where it is:

- **`openinference.span.kind`** is the central classifier (chapter 02 §"OpenInference semantic conventions"). It drives per-kind rendering in Phoenix.
- **`llm.input_messages.<i>.message.*`** and **`llm.output_messages.<i>.message.*`** are the indexed-sub-attribute shape; this is what lets a backend that speaks OpenInference render message lists.
- **`llm.tool_calls.<i>.tool_call.function.*`** lives on the LLM span (where the model emitted the call), not the TOOL span. The TOOL span uses the generic `input.value` / `output.value` + `tool.name` / `tool.description` set.
- **Top-of-tree `input.value` / `output.value`** on the AGENT root carries the user's raw question and the final reply — a legibility convenience that drives top-down walk (chapter 04 §"Prompt / response granularity").

### 2.3 `compare.md` — the actual deliverable

The code is scaffolding; `compare.md` is the produced artefact. Reference sketch follows.

```markdown
# compare.md — OTel GenAI vs OpenInference on support_qa

## Attribute-name diff table

| Concept               | OTel GenAI name                                   | OpenInference name                                   | Notes                                                                 |
|-----------------------|---------------------------------------------------|------------------------------------------------------|-----------------------------------------------------------------------|
| System / provider     | `gen_ai.system`                                   | `llm.provider`                                       | Same value, different key. Safe to mirror.                            |
| Requested model       | `gen_ai.request.model`                            | — (set `llm.model_name` with request value, or use `llm.invocation_parameters`) | OpenInference folds request/served into `llm.model_name`; set it twice — before/after — to distinguish, or mirror `gen_ai.request.model`. |
| Served model          | `gen_ai.response.model`                           | `llm.model_name`                                     | Mirror both for drift catching (chapter 05, model-version-drift signature). |
| Prompt tokens         | `gen_ai.usage.input_tokens`                       | `llm.token_count.prompt`                             | Same integer, different key.                                          |
| Completion tokens     | `gen_ai.usage.output_tokens`                      | `llm.token_count.completion`                         | Same integer, different key.                                          |
| Finish reason         | `gen_ai.response.finish_reasons` (list)           | — (no direct equivalent; emit as `app.finish_reasons` or on the OTel status) | OpenInference leans on OTel status + `output.value`; mirror the array under `gen_ai.*`. |
| Tool name             | `gen_ai.tool.name` + `gen_ai.operation.name=execute_tool` | `tool.name` + `openinference.span.kind=TOOL`         | Different classifiers, same meaning.                                  |
| Tool arguments        | On a child tool span (`gen_ai.tool.call.*`) or on the LLM span as the message turn | On LLM span: `llm.tool_calls.<i>.tool_call.function.arguments`; on TOOL span: `input.value` | OpenInference records the arguments twice — once as model output, once as tool input. |
| Message content       | Span events: `gen_ai.user.message` / `gen_ai.assistant.message` / `gen_ai.system.message` / `gen_ai.choice` | Indexed attributes: `llm.input_messages.<i>.message.content` / `llm.output_messages.<i>.message.content` | Shape difference — not just a rename. Viewers render these very differently. |
| Tool return           | On the tool child span (`gen_ai.tool.call.result` or convention event) | `output.value` on the TOOL span                      | OpenInference's TOOL kind is first-class.                             |

## Rendering diff

Observed in Arize Phoenix running on port 6006 against both services:

- `"Where is order 42?"` — **OpenInference**: three chat bubbles (system, user, assistant) under `llm.plan`, a tool-call widget showing `lookup_order({"order_id": "42"})`, a TOOL span with `input.value` / `output.value` panels, then the second LLM span with the final answer. **OTel GenAI**: the LLM span renders model / tokens / finish reasons cleanly, but the message content sits under a "Span events" tab as `gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.choice` — one row per event, content in a `content` attribute. Legible but one more click away.
- `"What's the ETA on 87?"` — same shapes; Phoenix shows the retrieval of `{"status":"processing","eta":"2026-11-12"}` as a JSON panel in the OpenInference run and as the `gen_ai.tool.call.result` event body in the OTel GenAI run.
- `"Do you ship to Iceland?"` — no tool call. Both variants show one LLM span with a plain text answer. OpenInference's `output.value` on the root makes the top-level "what did we say back" obvious; OTel GenAI requires drilling into the LLM span's `gen_ai.choice` event.
- `"Hi there."` — identical shape to the Iceland case; `output.value` win repeats.
- `"Tell me a joke about printers."` — model returned a short reply; the OTel GenAI variant correctly carries `gen_ai.response.finish_reasons=["stop"]`, which the OpenInference variant does not have an equivalent of (leaned on OTel status instead).

## Which one I would keep, why, and where I would mirror

For this surface, I would keep **OpenInference as primary** and **mirror `gen_ai.system` / `gen_ai.request.model` / `gen_ai.response.model` / `gen_ai.usage.*` / `gen_ai.response.finish_reasons`** on every LLM span. Reason: the primary workflow (chapter 05's bisection) spends most of its time reading message content and tool calls, and the OpenInference indexed-attribute shape renders directly in Phoenix without a second click. Finish reasons and request / response model are where OpenInference is quieter — I want to know when a provider served a different model than I asked for, and OTel GenAI's attribute names make that explicit.

If the backend posture flipped — say the organisation committed to a vendor that renders OTel GenAI events natively and does not understand OpenInference — I would flip the primary and keep the OpenInference indexed messages as the mirror, so the content is still on an attribute for anything downstream (mod-105 / mod-107) that reads spans without the viewer in the loop. In either case, the rule from chapter 02 holds: pick one primary per span, mirror the other where interop matters, never emit half-and-half.
```

Writing the dual variant (`app_dual.py`) is the stretch goal and uses the chapter 02 §"Example: dual-convention on the same LLM span" pattern verbatim — set both names to the same value on the same span.

## 3. Validation steps

A reviewer runs the solution as follows:

1. **Start a trace viewer**: `docker run -p 6006:6006 -p 4317:4317 arizephoenix/phoenix:latest` (check the current tag on <https://docs.arize.com/phoenix>).
2. **Install SDKs**: `pip install openai opentelemetry-api opentelemetry-sdk opentelemetry-exporter-otlp-proto-http`.
3. **Set the environment**: `export OPENAI_API_KEY=sk-…; export OTEL_EXPORTER_OTLP_ENDPOINT=http://localhost:6006`.
4. **Run each variant**: `python app_otel_genai.py` and `python app_openinference.py`. Both should flush cleanly; traces appear in Phoenix within ~1 s of the final line.
5. **Grep-check attribute coverage** on the exported traces (Phoenix supports JSON export):
   - OTel GenAI variant must carry all of: `gen_ai.system`, `gen_ai.operation.name`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.request.temperature`, `gen_ai.request.max_tokens`, `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.finish_reasons`, `gen_ai.response.id`, `gen_ai.tool.name`, `gen_ai.tool.call.id` — plus events `gen_ai.user.message`, `gen_ai.assistant.message`, `gen_ai.system.message`, `gen_ai.choice`.
   - OpenInference variant must carry all of: `openinference.span.kind`, `llm.provider`, `llm.model_name`, `llm.invocation_parameters`, `llm.input_messages.<i>.message.role/content`, `llm.output_messages.<i>.message.role/content`, `llm.token_count.prompt/completion/total`, `llm.tool_calls.<i>.tool_call.function.name/arguments`, `input.value` / `output.value`, and a `TOOL`-kind child span with `tool.name` / `tool.description`.
6. **Confirm the trace tree shape under both variants**: one `support_qa.answer` root; for tool-triggering inputs, two LLM children bracketing a tool child; for non-tool inputs, one LLM child only.
7. **Read `compare.md`** and check the three sections are present and non-narrative (table, bulleted rendering diff, two keep-and-mirror paragraphs).

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Both scripts run end-to-end on all five inputs and produce traces in the viewer. | Traces for every input visible. | Any variant crashes, hangs, or drops traces silently. |
| R2 | Every OTel GenAI attribute in the requirement list is present on the right span kind. | `grep` hits for all listed `gen_ai.*` names. | Any missing. |
| R3 | Every OpenInference attribute in the requirement list is present on the right span kind. | `grep` hits for all listed `llm.*`, `openinference.span.kind`, `tool.*`, `input.value`, `output.value`. | Any missing. |
| R4 | OTel GenAI carries message content on **events**; OpenInference carries it on **indexed attributes**. | Event names match `gen_ai.*.message` + `gen_ai.choice`; indexed keys match `llm.input_messages.<i>.message.content`. | Wrong shape or mixed. |
| R5 | Tool call and tool execution are first-class child spans under both variants. | `SpanKind.INTERNAL` + `gen_ai.operation.name=execute_tool` (GenAI) / `openinference.span.kind=TOOL` (OI). | Flattened into the LLM span. |
| R6 | No auto-instrumentor is installed (chapter 02 rule for this exercise). | No `openinference-instrumentation-*` / `opentelemetry-instrumentation-openai` in `requirements.txt`. | Any present. |
| R7 | `compare.md` contains the diff table with all ten rows, a per-input rendering diff, and the keep-and-mirror paragraphs. | All three sections present. | Any section missing or substituted with prose. |
| R8 | No secret is committed; `OPENAI_API_KEY` is read from env. | `os.environ[...]` pattern; no literal keys in source. | Hard-coded key found. |
| R9 | Content-size caps respected (no attribute larger than ~64 KB — chapter 04 rule of thumb). | Per-attribute sizes under cap for every trace. | Any oversized attribute. |
| R10 | Keep-and-mirror decision is grounded in the viewer observation, not vendor marketing. | Paragraph quotes the viewer behaviour ("content under 'events' tab", "indexed attrs rendered as bubbles"). | Generic "OpenInference is richer" claim with no observed evidence. |

A passing submission is 8+ of 10 with R1, R4, R7, R8 all hard-required.

## 5. Common mistakes

- **Mixing conventions on the same span.** Setting both `gen_ai.response.model` and `llm.model_name` without acknowledging the mirror. Chapter 02 permits this *when deliberate*; this exercise is about seeing each shape alone, so mix only in the stretch `app_dual.py` with a code comment that names the pattern.
- **Putting content twice on the OpenInference variant** — once on `llm.input_messages.*` and again on `input.value` for every message. The rule in chapter 04: top-of-tree `input.value` carries the *user's raw question*; indexed `llm.input_messages.*` carries the full chat context. Do both but do not duplicate the full message list.
- **Forgetting `gen_ai.response.finish_reasons`.** It is the one GenAI attribute OpenInference does not mirror cleanly, and the trace-viewer workflow (chapter 05) leans on it for loop-not-terminated signatures. Easy miss.
- **`SpanKind.SERVER` on the LLM call.** The model call is a `CLIENT` from the service's perspective; `SERVER` would reverse the causal direction (see the OTel SpanKind spec).
- **Hard-coding `max_tokens=4096` or `temperature=0.7`.** Budget guard: use `max_tokens=256` and `temperature=0.2` for the five-input run to keep spend inside the chapter-02 budget.
- **Using an auto-instrumentor "to save time."** You will fail R6, and the diff loses its point — the auto-instrumentor hides the attribute you were supposed to set by hand.
- **Not force-flushing** the SDK before `sys.exit`. Traces will drop silently.
- **Writing `compare.md` from marketing pages**, not the viewer. R10 catches this.
- **Narrating "the model confused the two conventions."** It did not; you did. Quote the attribute.

## 6. References

Official standards and project docs cited in this solution:

- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/> (status: development / experimental; re-read on each release cycle, per chapter 02).
- OpenTelemetry semantic conventions repo, <https://github.com/open-telemetry/semantic-conventions>.
- OpenTelemetry SDK — trace API (SpanKind, span events, span links, status codes), <https://opentelemetry.io/docs/specs/otel/trace/api/>.
- OpenTelemetry Protocol (OTLP), <https://opentelemetry.io/docs/specs/otlp/>.
- OpenInference specification, <https://github.com/Arize-ai/openinference> (spec under `spec/`; vocabulary values for `openinference.span.kind` and per-kind attribute lists).
- Arize Phoenix docs, <https://docs.arize.com/phoenix> (viewer behaviour for OpenInference and OTel GenAI; container tag on the docs landing page).
- Local module chapters: `lessons/mod-102-trace-instrumentation/02-otel-genai-and-openinference.md` (attribute tables and dual-annotation pattern).
- Local module chapter: `lessons/mod-102-trace-instrumentation/05-bisecting-a-bad-run.md` (motivation for keeping `gen_ai.response.finish_reasons` as a mirror).
