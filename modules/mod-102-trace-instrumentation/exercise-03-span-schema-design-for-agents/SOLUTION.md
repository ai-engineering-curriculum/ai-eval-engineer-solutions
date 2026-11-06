# exercise-03 — Span-Schema Design for Agents — Solution

> Reference solution for `lessons/mod-102-trace-instrumentation/exercises/exercise-03-span-schema-design-for-agents.md`.
> Chapters 02 and 04 are the primary input. All attribute names below are either cited from the OpenInference spec
> (<https://github.com/Arize-ai/openinference>), from the OpenTelemetry GenAI semantic conventions
> (<https://opentelemetry.io/docs/specs/semconv/gen-ai/>), or namespaced as `app.*` / `experiment.*` / `feature_flag.*`
> extensions per the chapter-02 rule.

## 1. Solution overview

The exercise produces a **span-attribute schema document** (`SCHEMA.md`) for a nested agent, plus a baseline-diff and a missing-SLI tracker. It is a *design* exercise — the output is the contract downstream modules (mod-106 CI, mod-107 online eval, mod-109 review routing, mod-110 lineage) will read against.

The reference solution picks the **support agent** (chapter 04's worked example) as the surface so the diff from the chapter 04 baseline is small and teachable; swapping in another surface changes rows, not sections. Four load-bearing design choices anchor it:

1. **OpenInference as the primary convention** (chapter 02 §"How the two conventions relate today"). Every LLM / TOOL / RETRIEVER / GUARDRAIL span is classified by `openinference.span.kind`; `gen_ai.*` attributes are present as **mirrors** on LLM spans for interop (chapter 02 §"Example: dual-convention on the same LLM span").
2. **Full-fidelity content lives on the LLM span it belongs to; the AGENT parent carries a redacted `output.value` summary** (chapter 04 §"Prompt / response granularity"). This is the single most common bug the schema prevents.
3. **The nine root attributes are non-negotiable** (chapter 04 §"The nine cross-cutting root attributes"). Every SLI downstream depends on them; MUST-enforcing them in mod-106 is the long-term stability play.
4. **`schema.version` is emitted from day one** with semver, a stated additive-vs-breaking rule, and a deprecation window. Chapter 04 is explicit that schemas evolve; building versioning in late is a mod-110 migration.

## 2. Worked implementation

The reference artefact is `SCHEMA.md` with eight sections. The full document follows; a condensed baseline-diff and a missing-SLI ledger sit at the bottom.

### 2.1 `SCHEMA.md`

```markdown
# SCHEMA.md — support_agent span-attribute schema

Primary convention: **OpenInference** (`openinference.span.kind`). Mirror set: core
OTel GenAI identity attributes (`gen_ai.system`, `gen_ai.request.model`,
`gen_ai.response.model`, `gen_ai.usage.*`, `gen_ai.response.finish_reasons`) on LLM
spans only. Local extensions under `app.*` / `experiment.*` / `feature_flag.*`.

## Section 1 — Surface context

`support_agent` is a customer-support surface: user sends a message, the agent plans
(LLM), retrieves (vector + keyword hybrid over our docs KB), optionally opens a
ticket (one `open_ticket` tool against Zendesk), replies (LLM), and passes through
an output guardrail. Expected traffic shape: ~1,000 requests / minute at peak,
~200 / minute mean. Framework is homegrown (no LangChain). Chosen convention:
OpenInference primary with GenAI mirror on identity attributes.

- `app.surface` = `support_agent`
- `service.name` = `support_agent`

## Section 2 — Span-kind inventory

| Kind       | Count per interaction | Wraps                                     | Notes                                             |
|-----------:|-----------------------|-------------------------------------------|---------------------------------------------------|
| AGENT      | 1 (root)              | The whole coordinator loop                | Always one per interaction.                       |
| LLM        | 2                     | `plan` + `reply` model calls              | Will grow to 3 if we add a judge (deferred).       |
| RETRIEVER  | 1                     | Hybrid search against docs KB             | 0 if planner decides no retrieval is needed.      |
| TOOL       | 0–1                   | `open_ticket` (Zendesk)                   | At most one per interaction by policy.            |
| GUARDRAIL  | 1                     | Output policy check                       | Always runs post-reply.                           |
| CHAIN      | — (omitted)           | —                                         | No named sub-pipelines yet; adding them later is additive. |
| EMBEDDING  | — (omitted)           | Embedded inside the RETRIEVER today       | If we split, we will emit per-call; additive bump. |
| RERANKER   | — (omitted)           | Not in the pipeline today                 | Chapter 04 stretch goal path; additive bump.      |
| EVALUATOR  | 0–many                | mod-107 judge scores                      | Attached post-hoc; schema already supports this.  |

## Section 3 — Per-kind attribute tables

### AGENT (root)

| MUST                                                               | SHOULD                                              | SHOULD-NOT                                           | Example value                                            |
|--------------------------------------------------------------------|-----------------------------------------------------|------------------------------------------------------|----------------------------------------------------------|
| `openinference.span.kind = "AGENT"`                                | `metadata` — small JSON with channel / locale       | Full LLM prompt content on this span                 | `"AGENT"`                                                |
| `app.surface`                                                      | `tag.tags`                                          | Full LLM reply content on this span (use `output.value` summary) | `"support_agent"`                               |
| `app.step = "agent.support_agent"`                                 | `experiment.arm`                                    | Tool arguments (lives on the TOOL span)              | `"agent.support_agent"`                                  |
| `session.id`                                                       | `feature_flag.<name>`                               | Retrieved document content (lives on RETRIEVER)      | `"sess_9e4b7f2a"`                                        |
| `user.id` — HMAC-SHA256 with per-tenant salt, prefix `"u_"`         | `app.cost_usd` (roll-up over children)              | Duplicating child-span data                          | `"u_3c8e1a04d5f06b22"`                                  |
| `service.version` — deploy marker                                  | `app.ttft_ms` / `app.tpot_ms` roll-ups (streaming)  |                                                      | `"2026.11.05-abc1"`                                       |
| `schema.version` — semver                                          |                                                     |                                                      | `"1.0.0"`                                                 |
| `input.value` — the user's raw message                             |                                                     |                                                      | `"Where is my order 12345?"`                              |
| `output.value` — redacted reply summary                            |                                                     |                                                      | `"Ticket #1247 opened; status update sent."`              |

### LLM

| MUST                                                              | SHOULD                                              | SHOULD-NOT                                                   | Example value |
|-------------------------------------------------------------------|-----------------------------------------------------|---------------------------------------------------------------|---------------|
| `openinference.span.kind = "LLM"`                                 | `llm.invocation_parameters` (JSON)                  | Retrieved chunk content — belongs on RETRIEVER                | `"LLM"`       |
| `llm.model_name` — served                                         | `llm.prompt_template.template` **or** `app.prompt_template.hash` | Tool return payloads — belong on TOOL                                    | `"gpt-4o-2024-08-06"` |
| `llm.provider`                                                    | `llm.prompt_template.variables`                      | Human-friendly span labels only ("plan") — also carry `app.step` | `"openai"`    |
| `llm.input_messages.<i>.message.role`                             | `llm.tools` (JSON of tool schemas passed in)         |                                                               | `"system"` / `"user"` / `"assistant"` / `"tool"` |
| `llm.input_messages.<i>.message.content`                          | `llm.tool_calls.<i>.tool_call.function.name/arguments` on tool-call turns |                                                     | `"Where is my order 12345?"` |
| `llm.output_messages.<i>.message.role/content`                    | `app.ttft_ms`, `app.tpot_ms` (streaming only)        |                                                               |               |
| `llm.token_count.prompt` / `.completion` / `.total`               | `app.cost_usd` (per-call)                            |                                                               | `842 / 117 / 959` |
| `gen_ai.response.finish_reasons` — mirror; shape `string[]`       |                                                     |                                                               | `["stop"]`    |
| `gen_ai.system` — mirror                                          |                                                     |                                                               | `"openai"`    |
| `gen_ai.request.model` / `gen_ai.response.model` — mirrors         |                                                     |                                                               |               |
| `gen_ai.usage.input_tokens` / `.output_tokens` — mirrors           |                                                     |                                                               |               |
| `app.step` — stable step id (`plan`, `reply`)                      |                                                     |                                                               | `"plan"`      |

### TOOL

| MUST                                                              | SHOULD                                | SHOULD-NOT                               | Example value                                          |
|-------------------------------------------------------------------|---------------------------------------|------------------------------------------|--------------------------------------------------------|
| `openinference.span.kind = "TOOL"`                                | `tool.description`                    | Full LLM prompt / completion              | `"TOOL"`                                              |
| `tool.name`                                                       | `tool.parameters` (JSON schema)       | Retrieval scores                         | `"open_ticket"`                                       |
| `input.value` — parsed arguments                                  | `app.step`                            |                                          | `{"order_id":"12345","reason":"no_shipping_update"}`  |
| `output.value` — tool return                                      | HTTP `url`, `status_code` where applicable |                                     | `{"ticket_id":1247,"status":"open"}`                  |
| OTel span `status` — `Error` on failure                           |                                       |                                          | `OK` / `ERROR`                                        |

### RETRIEVER

| MUST                                                              | SHOULD                                              | SHOULD-NOT                      | Example value        |
|-------------------------------------------------------------------|-----------------------------------------------------|---------------------------------|-----------------------|
| `openinference.span.kind = "RETRIEVER"`                           | `retrieval.documents.<i>.document.metadata`         | Full LLM reply                  | `"RETRIEVER"`        |
| `input.value` — the retrieval query                               | `app.retriever_name`                                | Tool return                     | `"order 12345 status"` |
| `retrieval.documents.<i>.document.id`                             | `app.top_k`                                         |                                 | `"doc-142"`          |
| `retrieval.documents.<i>.document.content` — capped (§5)           | `app.hybrid_alpha`                                  |                                 | `"Shipping updates…"`|
| `retrieval.documents.<i>.document.score`                          |                                                     |                                 | `0.81`               |

### GUARDRAIL

| MUST                                                              | SHOULD                             | SHOULD-NOT                      | Example value   |
|-------------------------------------------------------------------|------------------------------------|---------------------------------|-----------------|
| `openinference.span.kind = "GUARDRAIL"`                           | `app.guardrail_rule_hit`           | LLM prompt / completion dupes   | `"GUARDRAIL"`   |
| `app.guardrail_id` (stable id of the rule set)                    | OTel span `status`                 |                                 | `"policy_v14"`  |
| `app.guardrail_version` (semver of the ruleset)                   |                                    |                                 | `"14.2"`        |
| `app.guardrail_verdict` — one of `allow`, `rewrite`, `block`      |                                    |                                 | `"allow"`       |
| `input.value` — the candidate reply                               |                                    |                                 | `"Ticket #…"`   |
| `output.value` — the final reply (may equal input when `allow`)   |                                    |                                 | `"Ticket #…"`   |

### EVALUATOR (post-hoc, from mod-107)

| MUST                                                     | SHOULD                              | SHOULD-NOT                          | Example value            |
|----------------------------------------------------------|-------------------------------------|-------------------------------------|---------------------------|
| `openinference.span.kind = "EVALUATOR"`                  | `app.judge_rubric` — stable id      | Full prompts / reply (link, don't dupe) | `"EVALUATOR"`           |
| `app.judge_name`                                         | `app.judge_version`                 |                                     | `"groundedness_v1"`       |
| `app.judge_model`                                        |                                     |                                     | `"gpt-4o-2024-08-06"`     |
| `app.judge_score` (numeric, rubric-scaled)               |                                     |                                     | `4`                       |
| `app.judge_scored_span_id` — the span this score attaches to |                                 |                                     | `"0x7a9f…"`               |

Total MUST rows across kinds: **~34** (chapter 04 says keep under ~40). Good.

## Section 4 — The nine cross-cutting root attributes

| # | Attribute                                            | Value shape                            | Example                            |
|---|------------------------------------------------------|----------------------------------------|------------------------------------|
| 1 | `session.id`                                         | opaque stable per-conversation id      | `"sess_9e4b7f2a"`                  |
| 2 | `user.id`                                            | `"u_" + HMAC-SHA256(raw, salt)[:16]`   | `"u_3c8e1a04d5f06b22"`             |
| 3 | `app.surface` / `service.name`                       | kebab or snake, stable                 | `"support_agent"`                  |
| 4 | `service.version`                                    | deploy marker `YYYY.MM.DD-<shortsha>`  | `"2026.11.05-abc1"`                |
| 5 | `llm.model_name` (+ `gen_ai.response.model` mirror)  | provider's served model id             | `"gpt-4o-2024-08-06"`              |
| 6 | `app.prompt_template.hash`                           | `"sha256:…"` over the rendered template | `"sha256:0f7e89…"`                 |
| 7 | `experiment.arm` / `feature_flag.<name>`             | opaque short id                        | `experiment.arm = "reply-model-v3"` |
| 8 | `app.cost_usd`                                       | USD, sum over LLM + tool costs         | `0.0042`                           |
| 9 | `app.ttft_ms`, `app.tpot_ms` (streaming); span duration | ms                                 | `app.ttft_ms = 512`                |

If an SLI below depends on a row here that is absent, the schema is incomplete.

## Section 5 — Content-granularity rule

- **Full-fidelity content** (user message, system prompt, tool arguments, retrieved chunk content, LLM reply) lives on **the span the content belongs to** — never duplicated upward. The LLM span owns its full `llm.input_messages` and `llm.output_messages`. The TOOL span owns parsed `input.value` / `output.value`. The RETRIEVER span owns `retrieval.documents.<i>.document.content`.
- **The AGENT parent** carries:
  - `input.value` — the user's raw message (short; always under cap).
  - `output.value` — a **redacted reply summary** (one short sentence; e.g., `"Ticket #1247 opened; status update sent."`), **not** the full reply.
- **Size cap**: no single string attribute exceeds **65_536 bytes** (~64 KB — chapter 04 rule of thumb). When the payload would exceed the cap:
  - Upload the full value to blob storage at a content-addressable key (e.g., S3 `s3://support-agent-traces/blobs/sha256/<hex>`).
  - On the span, set:
    - `content.truncated = true`
    - `content.sha256 = "<hex>"`
    - `content.url = "s3://…"` (presigned URL for rehydration lives in the Collector, not in the span)
    - Keep a preview of the first ~1 KB on the original attribute so the viewer renders something.
- **The cap applies per attribute**, not per span. A RETRIEVER span with ten documents may be large across attributes; none of them individually exceeds the cap.

## Section 6 — SLI-to-attribute mapping

From the `mod-101/exercise-01` criteria table for `support_agent`:

| SLI                                       | Attribute expression                                                                                                             |
|-------------------------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| Task success — correct ticket opened      | `tool.open_ticket.input.value` ⋈ expected ticket schema from the eval fixture                                                     |
| Groundedness — reply cites retrieved content | `llm.reply.llm.output_messages.0.message.content` ⋈ `retriever.docs.search.retrieval.documents.*.document.content`               |
| Refusal correctness (per family)          | `guardrail.output_policy.app.guardrail_verdict` per expected family                                                               |
| Cost per resolved conversation            | Root `app.cost_usd` summed per `session.id` resolved                                                                              |
| End-to-end p95 latency                    | Root span duration (OTel built-in) filtered by `app.surface` and `service.version`                                                |
| TTFT (streaming)                          | Root `app.ttft_ms` (==`llm.reply.app.ttft_ms`)                                                                                   |
| Model-version attribution                 | Root `gen_ai.response.model` (or `llm.model_name`) grouped per week                                                               |
| Experiment arm attribution                | Root `experiment.arm`                                                                                                             |

Every row maps; `missing-slis.md` is empty for v1.0.0 of the schema.

## Section 7 — User-id and PII handling

- `user.id` is stored as `"u_" + HMAC-SHA256(raw_user_id, tenant_salt).hex()[:16]`. **HMAC-SHA256 is specified by NIST FIPS 198-1** (<https://csrc.nist.gov/publications/detail/fips/198/1/final>). The per-tenant salt lives in a KMS; a small production-only hashing service holds the salt in memory at process start and never logs it. The hash is **pseudonymous** under GDPR Art. 4(5) (<https://eur-lex.europa.eu/eli/reg/2016/679/oj>), not anonymous — personal-data obligations still apply, and the residual privacy risk is enumerated in the surface's governance plan (mod-112).
- **Scrubbed at the application** (before `set_attribute`): any attribute key in the following list, with the stated redaction pattern. The full policy runtime is exercise-05; this document fixes the keys only.
  - `input.value` (regex redaction for email, phone, credit-card-like digit strings).
  - `output.value` (same).
  - `llm.input_messages.*.message.content`, `llm.output_messages.*.message.content` (same).
  - `tool.open_ticket.output.value` — strip `customer_email` and `phone` fields before `set_attribute`.
- **Scrubbed at the Collector** (defense in depth; policy runtime in exercise-05):
  - Mirror of application scrubs via `attributesprocessor`.
  - One additional PII sweep via `redactionprocessor` using Presidio-sourced regex patterns.
  - Hard denylist on any `app.raw_*` keys as a safety net.

## Section 8 — Versioning and deprecation

- `schema.version` is **semver** (`MAJOR.MINOR.PATCH`), emitted on every AGENT root.
- Launch version: **`1.0.0`**.
- **Additive** (MINOR bump): new attribute key, new SHOULD on an existing kind, new span kind. Example: adding `RERANKER` with `retrieval.documents.<i>.document.rerank_score` is `1.0.0 → 1.1.0`.
- **Breaking** (MAJOR bump): renamed attribute, changed value shape, moved attribute between spans, removed MUST. Example: renaming `app.cost_usd` to `app.cost.usd` is `1.0.0 → 2.0.0` and triggers a downstream migration in mod-107 / mod-110.
- **Deprecation window**: on a breaking change, emit **both** the old and new attribute names simultaneously for one calendar quarter, with the old name tagged as deprecated in the schema doc. After the window, the old name is removed. Mod-110's schema-registry entry carries the window metadata.

End of SCHEMA.md.
```

### 2.2 `schema-diff-from-baseline.md`

```markdown
# Diff vs chapter-04 baseline (support_agent)

The schema is intentionally close to the chapter-04 baseline — the surface is the same. Divergences:

- **CHAIN kind omitted.** Chapter 04 lists CHAIN as 0–3 per interaction; our planner / RAG / reply loop does not have named sub-pipelines yet. Adding CHAIN is additive (MINOR bump); we will revisit after exercise-04's walkthroughs surface a real need.
- **Mirror attributes are MUST on LLM, not SHOULD.** Chapter 04 presents the GenAI mirror as a dual-annotation example; we make the identity mirror (`gen_ai.system`, `gen_ai.request.model`, `gen_ai.response.model`, `gen_ai.usage.*`, `gen_ai.response.finish_reasons`) MUST so model-version-drift detection (chapter 05) does not fail on backends that render the GenAI shape.
- **EVALUATOR MUSTs include `app.judge_scored_span_id`.** Chapter 04 does not call this out specifically. We make it MUST so mod-107's judge can attach a score back to the right LLM / TOOL span unambiguously; without it, judge scores float.
- **Content-cap exact value is 65_536 bytes.** Chapter 04 suggests ~64 KB; we fix 65_536 as the operational cap so the application wrapper has a single constant to enforce.
- **`service.version` format fixed to `YYYY.MM.DD-<shortsha>`.** Chapter 04 shows this shape as an example; we adopt it as policy so attribution queries (chapter 05 §"bisect by attribute") are deterministic.
- **No RERANKER kind.** Our retriever is hybrid but does not run a separate rerank step. When it does, we will add RERANKER as an additive bump and the chapter-04 baseline covers the attribute names.

Nothing in the chapter-04 baseline is removed; every divergence is additive or a tightening.
```

### 2.3 `missing-slis.md`

```markdown
# Missing SLIs — v1.0.0

For schema v1.0.0, every SLI in `mod-101/exercise-01`'s criteria table for
`support_agent` maps to a Section-6 row. The list is empty.

If this file is non-empty in a future version:

- Format each entry as `- SLI name: smallest schema change that would close the gap.`
- A non-empty list blocks the schema from reaching release until mod-101 agrees the SLI is non-measurable or the schema patch ships.
```

## 3. Validation steps

1. **Round-trip every SLI**: read each row of Section 6 aloud. Example: *"Groundedness is computed by comparing `llm.reply.llm.output_messages.0.message.content` against `retriever.docs.search.retrieval.documents.*.document.content`."* Chapter 04 says: if you can't say the sentence, the mapping is wrong.
2. **Attribute-name provenance**: for every attribute in Section 3, confirm it either:
   - Appears in the OpenInference `spec/` directory, or
   - Appears in the OTel GenAI semconv (<https://opentelemetry.io/docs/specs/semconv/gen-ai/>), or
   - Is under `app.*` / `experiment.*` / `feature_flag.*` / `service.*` / `session.*` / `user.*` / `schema.*` / `content.*`.
3. **MUST count check**: total MUSTs across kinds ≤ 40 (chapter 04 rule). Current count is ~34.
4. **Root attribute completeness**: every row in Section 4 is present on the AGENT MUST list in Section 3.
5. **Content-cap concreteness**: Section 5 states an exact byte number and the four overflow fields (`content.truncated`, `content.sha256`, `content.url`, preview).
6. **Pseudonymity statement**: Section 7 names HMAC-SHA256, per-tenant salt in KMS, GDPR Art. 4(5), and the pseudonymous-not-anonymous distinction.
7. **Diff grounded in surface**: `schema-diff-from-baseline.md` names each deviation and *why*. Empty / "no diff" is not an acceptable answer when the surface is identical — at minimum, tightenings and fixed value shapes count as diffs.
8. **Document read-time ≤ 15 minutes** for a reader who has chapters 02 and 04 on hand (chapter 04 acceptance criterion).

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | All eight sections present and non-empty. | §1–§8 all filled. | Any section stubbed. |
| R2 | Every attribute name in Section 3 is citable to OpenInference / GenAI or namespaced under `app.*` / `experiment.*` / `feature_flag.*`. | Zero free-form names in spec namespaces. | Any invented `llm.foo` or `gen_ai.foo`. |
| R3 | Every mod-101 SLI maps to an attribute expression or is listed in `missing-slis.md`. | Full coverage. | An SLI is silently dropped. |
| R4 | Section 5 states full-content-on-LLM, parent-carries-summary, and a concrete cap + overflow procedure. | Exact byte number + four overflow fields. | Vague "we cap large strings." |
| R5 | Section 7 names HMAC-SHA256, per-tenant salt in KMS, GDPR Art. 4(5). | All three present. | SHA256 without HMAC, no salt, no pseudonymity note. |
| R6 | Section 8 names semver, additive-vs-breaking examples, and a deprecation window. | All three present. | Missing deprecation window. |
| R7 | `schema-diff-from-baseline.md` explains each divergence. | Row per divergence. | "Same as baseline" when the surface differs. |
| R8 | MUST count across kinds ≤ 40. | Count is bounded. | Over 40 MUSTs. |
| R9 | No LLM content on AGENT root; no tool return on AGENT root (Section 3 SHOULD-NOT columns respected by the examples). | Row-aligned SHOULD-NOTs. | SHOULD-NOT columns empty or ignored. |
| R10 | Document reads in ≤ 15 minutes. | A peer reviewer confirms. | Takes longer; usually a sign of duplicated content across sections. |

A passing submission is 8+ of 10 with R1, R3, R4, R5, R6 all hard-required.

## 5. Common mistakes

- **Inventing `llm.*` or `gen_ai.*` names.** Those namespaces belong to the specs; collide today or tomorrow. Chapter 02 is explicit: custom fields go under `app.*`.
- **Putting the full prompt on the AGENT root for convenience.** It will appear twice in the viewer and bloat storage. The parent gets a summary.
- **`user.id = SHA256(raw)` without a salt.** Chapter 04 §"User-id-or-hash" calls this an obvious look-up-table attack. The HMAC and the per-tenant salt are both required.
- **Section 6 rows mapped to "the LLM span."** An SLI needs an attribute *expression*, not a span name. "The LLM reply content is on `llm.reply.llm.output_messages.0.message.content`" is the shape.
- **Mixing MUST and SHOULD columns.** If a row is in MUST, mod-106 CI will enforce it on every trace; move freely to SHOULD when the cost outweighs the SLI.
- **Omitting `app.step`.** Chapter 04 pushes `app.step` as the stable step id; without it, LLM spans are only recognisable by their human labels, which drift with code renames.
- **Treating RERANKER / EMBEDDING as always-on.** Chapter 04 says drop kinds that don't apply; empty kinds are noise.
- **Writing a 25-page SCHEMA.md.** The acceptance criterion is 15-minute read time; the document loses value past that.
- **Not citing GDPR Art. 4(5) and FIPS 198-1 in Section 7.** The Governance sign-off (mod-112) looks for these; omitting them restarts the review.
- **"Deferred to exercise-05"** in Section 7's keys list. The *keys* are fixed in this exercise; only the *runtime* lives in exercise-05.

## 6. References

Official standards and project documentation:

- OpenInference specification, <https://github.com/Arize-ai/openinference> (spec under `spec/`; `openinference.span.kind` vocabulary and per-kind attribute lists).
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/>.
- OpenTelemetry semantic conventions repo, <https://github.com/open-telemetry/semantic-conventions>.
- OpenTelemetry SDK — trace API (SpanKind, span events, span links, status codes), <https://opentelemetry.io/docs/specs/otel/trace/api/>.
- NIST FIPS 198-1 — The Keyed-Hash Message Authentication Code (HMAC), <https://csrc.nist.gov/publications/detail/fips/198/1/final>.
- NIST SP 800-122 — Guide to Protecting the Confidentiality of Personally Identifiable Information, <https://csrc.nist.gov/publications/detail/sp/800-122/final>.
- GDPR — Regulation (EU) 2016/679, <https://eur-lex.europa.eu/eli/reg/2016/679/oj> (Art. 4(1) personal data; Art. 4(5) pseudonymisation).
- OWASP Top 10 for Large Language Model Applications, <https://owasp.org/www-project-top-10-for-large-language-model-applications/> (LLM06 drives the Section 7 scrubbing-keys list).
- NIST AI Risk Management Framework (AI 100-1), <https://www.nist.gov/itl/ai-risk-management-framework> (schema evidence feeds the Measure function in mod-112).

Local module chapters:

- `lessons/mod-102-trace-instrumentation/02-otel-genai-and-openinference.md` — attribute-name tables; dual-annotation pattern.
- `lessons/mod-102-trace-instrumentation/04-span-schema-design.md` — the nine root attributes, per-kind tables, prompt-granularity rule, content sizing, versioning.
