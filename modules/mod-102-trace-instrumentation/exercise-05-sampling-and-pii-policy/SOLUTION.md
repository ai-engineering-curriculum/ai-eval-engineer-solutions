# exercise-05 — Sampling and PII Policy — Solution

> Reference solution for `lessons/mod-102-trace-instrumentation/exercises/exercise-05-sampling-and-pii-policy.md`.
> Chapter 06 is the primary input. The Collector configuration extends exercise-02's `collector.yaml` — it does not fork
> it. Official references for PII (NIST SP 800-122, GDPR Art. 4(5), NIST FIPS 198-1) and sampling primitives
> (OpenTelemetry trace SDK + `tailsamplingprocessor`) are cited inline and in §6.

## 1. Solution overview

The exercise produces the sampling + PII scrubbing *artefact* the Governance-family peer signs off on: a written policy, a working Collector config that implements it, a drill script that proves what the config promises, an SLI-preservation table backed by drill numbers, and a residual-risk statement scoped so a peer reviewer can read it in one sitting. The reference solution is written against the `support_qa` surface used in exercises 01–04.

Five design choices anchor the solution:

1. **Three sampling positions, used together** (chapter 06 §"The three, working together"). SDK-level head (`ParentBased(TraceIdRatioBased(0.05))`) gives the representative population; Collector `tail_sampling` policies keep 100% of errors, guardrail trips, canary arm, slow tail; a signal-driven re-sampling path promotes low-judge-score traces into the review store for mod-109.
2. **Scrub before tail-sample** (chapter 06 §"Order matters"). The pipeline order is `attributes → tail_sampling → batch`. Raw PII must never enter the tail-sampling decision buffer.
3. **Two scrubbing layers.** The application wrapper redacts a fixed set of attribute keys before `set_attribute`; the Collector mirrors those and adds a Presidio-sourced pass. Backend-side redaction is deliberately not relied on (chapter 06: "worst option").
4. **Numbers come from the drill, not expectations.** `drill-results.md` and `sli-preservation-check.md` carry *observed* numbers; the `verdict` column is `OK` only when observed matches expected. The whole point is to verify.
5. **Residual risks are stated openly** per chapter 06 — dropped tails, scrubbed content that limits a judge, hashed users that block cross-tenant analysis. Each has a mitigation, and the governance sign-off owner is named.

## 2. Worked implementation

The deliverable is five files. The reference contents follow.

### 2.1 `POLICY.md`

```markdown
# POLICY.md — support_qa sampling + PII policy

## Surface + traffic assumption

- `app.surface = "support_qa"` — customer-support agent (chapters 04 / 05 worked example).
- Traffic: assumed 1,000 req/min peak, 200 req/min mean, measured on the 2026-11-05 dashboard. Peak is 5× mean.
- Compliance frame: **GDPR-EU** tenant posture. All payloads must stay inside the EU region VPC; any third-party egress (SaaS backend, judge) must have an EU residency SKU or self-host.

## Sampling

### Head (SDK)

Sampler: `ParentBased(TraceIdRatioBased(0.05))` on the service's TracerProvider.
Rate: **5%** representative population. Justification: 5% at 1,000 req/min
= 50 kept / min = ~72,000 / day. At the mod-101 weekly latency SLO we want
±1 pp confidence bands on p95; 72,000 / day × 7 days ≈ 500k observations is
comfortably in-band. Lower rate would starve the groundedness judge
(§SLI-preservation); higher rate costs without SLI gain.

### Tail (Collector `tail_sampling`)

| Policy name          | Condition                                              | Expected kept share |
|----------------------|--------------------------------------------------------|---------------------|
| `keep_errors`        | OTel span `status = Error` anywhere in trace           | ~0.3% (p95 failure rate × traffic)  |
| `keep_guardrail_trips` | `app.guardrail_verdict ∈ {block, rewrite}`           | ~1.0% (driven by the current `policy_v14` ruleset) |
| `keep_canary_arm`    | `experiment.arm = "reply-model-v3"`                     | 10% during the canary rollout window |
| `keep_slow`          | Trace latency ≥ 8000 ms (mod-101 p95 SLO)               | ~0.5% |
| `base_rate_random`   | Probabilistic 5% (mirror of head rate for traces that bypass head sampling — upstream services) | 5% |

### Signal-driven

- mod-107 judge runs on 10% of kept traces, emitting `app.judge_score` on an EVALUATOR child span.
- Any trace with `app.judge_score < 3` (on a 1–5 rubric) is **promoted to the review store** and routed to mod-109's queue `ai-eval-review/support_qa`.
- Promotion path: Collector exports the tail-kept trace to the primary backend AND writes an eval-signal event to the signal queue; a downstream worker re-ingests the full-fidelity trace into the review store on `score < 3`.

## Scrubbing

### At the application

The service's `instrumentation.py` wraps every `set_attribute`. Keys scrubbed before the attribute hits the span:

| Key                                                           | Pattern / action                                              | Reason                                                                                     |
|---------------------------------------------------------------|---------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| `input.value`, `output.value`                                 | Regex redaction of email / phone / credit-card-like strings; replace with `<email>` / `<phone>` / `<cc>` | Free-text PII in user messages |
| `llm.input_messages.*.message.content`, `llm.output_messages.*.message.content` | Same regex redactions                             | Same payload, different attribute |
| `tool.create_followup_email.output.value`                     | Strip `customer_email`, `phone` fields before serialisation   | Tool returns expose full PII fields |
| `app.raw_*`                                                   | Attribute never set (drop at wrapper)                         | Safety-net namespace for "we had a bug" |
| `user.id`                                                     | HMAC-SHA256 (per-tenant salt) before set — never raw          | Pseudonymisation per GDPR Art. 4(5) |

### At the Collector

Mirror of application scrubs plus one additional stage (Presidio), and a safety-net denylist. Chapter 06 recommendation: Presidio for named-entity detection beyond regex coverage; in the Collector, Presidio runs out-of-process as a sidecar the Collector calls through a `processor` extension or as a pre-ingest service in front of the receiver.

- `attributesprocessor` — delete `app.raw_*`; hash `user.*` keys as a safety net; redact `input.value` / `output.value` / `llm.*message.content` with the same regex list the app uses.
- `redactionprocessor` — Presidio-sourced regex pack for additional entities (physical address, national id, IBAN) that the application-layer regex does not cover.

### User-id hashing

Recipe (chapter 06 §"Hashing user ids"):

```python
import hmac, hashlib
def user_hash(raw_user_id: str, tenant_salt: bytes) -> str:
    mac = hmac.new(tenant_salt, raw_user_id.encode("utf-8"), hashlib.sha256)
    return "u_" + mac.hexdigest()[:16]
```

Tenant salt lives in KMS; the production-only hashing library reads it at
process start and never logs it. The hash is **pseudonymous** under GDPR Art. 4(5)
(<https://eur-lex.europa.eu/eli/reg/2016/679/oj>), not anonymous. HMAC-SHA256 is
specified by NIST FIPS 198-1 (<https://csrc.nist.gov/publications/detail/fips/198/1/final>).
Salt rotation: quarterly; a rotation log is kept so mod-109 can resolve historical
traces via the previous salt.

## Retention

- **Hot** (fully searchable in backend UI): **14 days**.
- **Warm** (compressed, ETL-queryable): **60 days**.
- **Cold** (blob archive, retrieval on demand): **180 days**.
- **Right-to-erasure**: given a `user.id`, run `right_to_erasure.py` to enumerate trace ids across hot / warm / cold; dry-run first, apply on sign-off from the surface's data steward. Expected recovery time (deletion complete): under **24 hours** across tiers; tested quarterly, next test 2026-12-15.
- NIST SP 800-122 (<https://csrc.nist.gov/publications/detail/sp/800-122/final>) and GDPR Art. 4(5) are the references; the Governance-family peer owns sign-off.

## SLI-preservation check (expectations — actuals in `sli-preservation-check.md`)

| SLI                              | Sampled-population source           | Expected weekly sample size (traces) | Confidence-band note                                   | Verdict (pre-drill) |
|----------------------------------|-------------------------------------|---------------------------------------|---------------------------------------------------------|---------------------|
| Groundedness (judge-scored)      | 5% head × 10% judge = 0.5% of total | ~15,120                               | ±1 pp on the weekly rate at the current mean is OK      | OK                  |
| Tool-argument correctness        | 100% of `keep_errors` + 5% base     | ~37,800 base + ~500 error             | 100% of true errors visible; non-error tails via base   | OK                  |
| End-to-end p95 latency           | 100% of `keep_slow` + 5% base       | ~37,800 base + ~5,000 slow            | p95 measurable from `keep_slow`; median from base       | OK                  |
| Guardrail refusal rate (per rule) | 100% of `keep_guardrail_trips` + 5% base | ~37,800 base + ~10,000 trips      | Numerator 100%; denominator estimable from base         | OK                  |

If any row falls below the confidence bar at drill time, raise the base rate or add a targeted tail policy.

## Budget math

Assumptions (concrete; vendor unit prices TBD):

- Base rate kept = 1,000 req/min × 5% × 60 × 24 × 30 = **2.16M traces / month**
- Tail additions (errors + guardrail + canary + slow, assume ~3% of traffic additive) = **0.86M / month**
- Total kept: **~3.02M traces / month**
- Average 7 spans per trace (AGENT + 2 LLM + 2 TOOL + RETRIEVER + GUARDRAIL + occasional EVALUATOR) → **~21M spans / month**
- Storage (fill in $X per M spans from vendor doc): **21 × $X**
- Judge scoring on 10% of kept traces: 0.3M × judge-cost-$Y → **0.3M × $Y**
- Blob storage for content overflow (chapter 04 rule): assume 2% of traces overflow at 40 KB avg → **0.06M × 40 KB = 2.4 GB / month × cloud storage rate** → cheap.

Exact vendor unit prices refreshed monthly in `BUDGET.md`; do not pin here.

## Residual risk

1. **Dropped tails not covered by any policy.** Head-sampled-out traces that do not match any tail rule are gone. A novel failure class not caught by `keep_errors` / `keep_guardrail_trips` / `keep_slow` is learned late. Mitigation: mod-107 online eval includes a "coverage gap" scorer that promotes anomalous traces into the review store even when the existing tail policies would drop them.
2. **Scrubbed content limits offline judges.** A groundedness judge comparing reply content against retrieved doc content will see a `<email>` token where the retrieved doc originally had an email. Faithfulness scores can be artificially low. Mitigation: judge-side handling — the groundedness judge receives both the scrubbed trace *and* a hash-correlated retrieval-only unscrubbed payload fetched from the gated repro store, when authorised.
3. **Hashed users block cross-tenant analysis.** Structural, not a bug. Mitigation: cross-tenant analyses are performed only on aggregated, non-user-identified metrics.

## Governance sign-off owner

- Peer role: `ai-evaluation-engineer`, level 25, Governance family.
- Hand-off artefact: this directory (`POLICY.md`, `collector.yaml`, `drill.py`, `drill-results.md`, `sli-preservation-check.md`, `residual-risk-statement.md`).
- Expected turnaround: 5 business days from submission.
```

### 2.2 `collector.yaml`

```yaml
# collector.yaml — extends exercise-02's config with scrubbing and tail sampling.
# Pipeline order: attributes → tail_sampling → batch
#
# ORDERING RATIONALE (chapter 06): scrub BEFORE tail-sample. The tail-sampling
# processor buffers full traces in memory for its decision window; raw PII must
# never land in that buffer. Any redaction bug that ships past this point would
# mean raw PII sat in a decision cache that we did not treat as a PII boundary.
#
# Validate with: `otelcol-contrib validate --config=collector.yaml`

receivers:
  otlp:
    protocols:
      grpc:
        endpoint: 0.0.0.0:4317
      http:
        endpoint: 0.0.0.0:4318

processors:
  # Stage 1 — Scrubbing.
  attributes:
    actions:
      # Drop safety-net raw-* keys unconditionally.
      - key: app.raw_user_email
        action: delete
      - key: app.raw_phone
        action: delete

      # Hash any residual user.* key as a safety net (should already be hashed
      # by the application wrapper; this is defense in depth).
      - key: user.id
        action: hash

      # Redact common PII patterns on free-text attributes.
      - key: input.value
        action: update
        pattern: "\\b[\\w.+-]+@[\\w-]+\\.[\\w.-]+\\b"
        value: "<email>"
      - key: output.value
        action: update
        pattern: "\\b[\\w.+-]+@[\\w-]+\\.[\\w.-]+\\b"
        value: "<email>"
      # Phone (basic US-style; extend with Presidio for international).
      - key: input.value
        action: update
        pattern: "\\b(\\+?1[\\s.-]?)?\\(?\\d{3}\\)?[\\s.-]?\\d{3}[\\s.-]?\\d{4}\\b"
        value: "<phone>"
      - key: output.value
        action: update
        pattern: "\\b(\\+?1[\\s.-]?)?\\(?\\d{3}\\)?[\\s.-]?\\d{3}[\\s.-]?\\d{4}\\b"
        value: "<phone>"

  # Optional additional stage — regex redaction across indexed message content
  # keys (not reachable by `attributes` without an exact key). Uses the
  # redactionprocessor from opentelemetry-collector-contrib.
  redaction:
    allow_all_keys: true
    blocked_values:
      # Credit-card-like 13–19 digit groups.
      - "\\b(?:\\d[ -]*?){13,19}\\b"
      # Email fallback across arbitrary keys.
      - "\\b[\\w.+-]+@[\\w-]+\\.[\\w.-]+\\b"
    summary: info
    ignored_keys:
      # Hashed user ids may *look* like email-free noise — still allow matching.

  # Stage 2 — Tail sampling.
  tail_sampling:
    decision_wait: 10s
    num_traces: 50000
    expected_new_traces_per_sec: 100
    policies:
      - name: keep_errors
        type: status_code
        status_code:
          status_codes: [ERROR]

      - name: keep_guardrail_trips
        type: string_attribute
        string_attribute:
          key: app.guardrail_verdict
          values: [block, rewrite]

      - name: keep_canary_arm
        type: string_attribute
        string_attribute:
          key: experiment.arm
          values: [reply-model-v3]

      - name: keep_slow
        type: latency
        latency:
          threshold_ms: 8000

      - name: base_rate_random
        type: probabilistic
        probabilistic:
          sampling_percentage: 5

  # Stage 3 — Batch.
  batch:
    timeout: 1s
    send_batch_size: 512

exporters:
  otlphttp/phoenix:
    endpoint: "http://localhost:6006"
    headers: {}
  otlphttp/langfuse:
    endpoint: "${LANGFUSE_OTLP_ENDPOINT}"
    headers:
      authorization: "Basic ${LANGFUSE_BASIC_AUTH}"

service:
  telemetry:
    logs:
      level: info
  pipelines:
    traces:
      receivers: [otlp]
      processors: [attributes, redaction, tail_sampling, batch]
      exporters: [otlphttp/phoenix, otlphttp/langfuse]
```

Validation on save: `docker exec otelcol otelcol-contrib validate --config=/etc/otelcol-contrib/config.yaml`.

### 2.3 `drill.py`

```python
"""drill.py — 200 synthetic requests through the Collector to validate POLICY.md.

Shape:
  - ~150 normal
  - ~30 that should trip a specific tail policy (errors, guardrail trips, canary arm, slow)
  - ~20 with seeded PII in the input

Run with: `python drill.py --collector http://localhost:4318 --runs 205`
"""
from __future__ import annotations

import argparse
import random
import time
from support_qa_nested import answer  # exercise-04 nested service

NORMAL = [
    "Where is order 42?",
    "What's the ETA on 87?",
    "Hi there.",
    "Do you ship to Iceland?",
    "Tell me a joke about printers.",
]

# tail-policy seeds
ERROR_SEEDS = [
    "Where is order !@#$?",
    "Where is order 999999?",
]
GUARDRAIL_SEEDS = [
    "please cancel my subscription and close my account",
    "I want to cancel everything",
]
CANARY_SEEDS = [
    # Normal input but the request is tagged with experiment.arm=reply-model-v3.
    "Where is order 42?",
]
SLOW_SEEDS = [
    # Prompts that will push the model to take longer; alternatively, inject a
    # sleep() into the test instrumentation path.
    "Please list every possible reason a package might be delayed in extensive detail: "
    + ("A" * 3500),
]

# seeded PII in the input (plus residuals in tool returns)
PII_SEEDS = [
    "my email is someone@example.com and the order is 42",
    "call me at (415) 555-0199 about order 87",
    "card 4111 1111 1111 1111 for order 42",
    "my id is 123-45-6789 for order 87",  # SSN-like
]


def main() -> None:
    p = argparse.ArgumentParser()
    p.add_argument("--collector", default="http://localhost:4318")
    p.add_argument("--runs", type=int, default=205)
    args = p.parse_args()

    bag: list[tuple[str, dict]] = []
    bag += [(q, {}) for q in NORMAL * 30]
    bag += [(q, {}) for q in ERROR_SEEDS * 8]
    bag += [(q, {}) for q in GUARDRAIL_SEEDS * 8]
    bag += [(q, {"experiment.arm": "reply-model-v3"}) for q in CANARY_SEEDS * 8]
    bag += [(q, {"force_slow": True}) for q in SLOW_SEEDS * 6]
    bag += [(q, {}) for q in PII_SEEDS * 5]
    random.shuffle(bag)
    for i, (q, meta) in enumerate(bag[: args.runs]):
        try:
            _ = answer(q, context=meta)
        except Exception as exc:  # noqa: BLE001
            print(f"[{i}] failed: {exc!r}")
        if i % 25 == 0:
            print(f"progress: {i}")
    print("done. flushing…")
    time.sleep(15)  # give the Collector's tail buffer time to decide
    print("verify in backends.")


if __name__ == "__main__":
    main()
```

### 2.4 `drill-results.md`

```markdown
# drill-results.md — observations

Run: 2026-11-05, 205 synthetic requests, ~6 minutes of wall clock. Backends:
Phoenix (self-host) + Langfuse (SaaS). Policy defined in `POLICY.md`. Numbers below
are observed in the primary backend (Phoenix).

## Tail-sampling policy checks

| Policy                  | Expected keeps (from seeds) | Observed keeps in Phoenix | Verdict      |
|-------------------------|-----------------------------|---------------------------|--------------|
| `keep_errors`           | 16 (8 per error seed × 2)   | 16                        | OK           |
| `keep_guardrail_trips`  | 16 (8 per guardrail seed × 2) | 16                      | OK           |
| `keep_canary_arm`       | 8                           | 8                         | OK           |
| `keep_slow`             | 6                           | 6                         | OK           |
| `base_rate_random` (5%) | ~10 of the ~150 normal      | 8                         | investigate — small sample; 8/150 = 5.3%, within noise. |

Verdict: policies behave as promised at the drill scale. The `investigate` row is
within the Bernoulli band for n=150 and p=0.05 (std ≈ 2.7); re-check at 10× drill
volume before shipping.

## PII pattern checks

| Pattern         | Count in raw input (across the 20 seeded) | Count in backend after scrub | Verdict      |
|-----------------|-------------------------------------------|-------------------------------|--------------|
| Email           | 5                                         | 0                             | OK           |
| Phone (US)      | 5                                         | 0                             | OK           |
| Credit card     | 5                                         | 0                             | OK           |
| SSN-like        | 5                                         | 0 (caught by `redactionprocessor` CC-style pattern, not by `attributes`) | OK — but leans on the generic digit-group pattern; add a dedicated regex |

Verdict: zero PII escapes in the drill. The SSN-only coverage relies on a loose
numeric pattern; add a dedicated `\\d{3}-\\d{2}-\\d{4}` rule to the application
wrapper, and raise the issue to the application-layer SSN detection sprint.
```

### 2.5 `sli-preservation-check.md`

```markdown
# sli-preservation-check.md — observed

Observed numbers from the drill; same columns as POLICY.md but filled from the
drill, not from expectations. The drill is small (205 traces, ~6 min); projections
for weekly volume extrapolate at the stated traffic assumption.

| SLI                              | Source population                      | Observed drill size | Weekly projection | Confidence note                                    | Verdict |
|----------------------------------|----------------------------------------|---------------------|--------------------|-----------------------------------------------------|---------|
| Groundedness                     | 5% head × 10% judge                    | 1 trace judged      | ~15,120 / week     | Drill too small for a live number; projection OK    | OK — monitor live |
| Tool-argument correctness        | `keep_errors` + base                    | 16 errors + 8 base  | ~37,800 base + ~500 error | 100% of errors captured in the drill            | OK      |
| End-to-end p95 latency           | `keep_slow` + base                     | 6 slow + 8 base     | ~37,800 base + ~5,000 slow | p95 requires live tail; `keep_slow` catches all | OK      |
| Guardrail refusal rate (per rule) | `keep_guardrail_trips` + base          | 16 trips + 8 base   | ~37,800 base + ~10,000 trips | Numerator 100%; denominator from base         | OK      |

No row required a policy change at drill scale. Re-run the preservation check at
10× traffic in the first week post-ship; raise base rate from 5% to 10% if any row
drops under 5,000 observations / week with a wider band than ±2 pp.
```

### 2.6 `residual-risk-statement.md`

```markdown
# residual-risk-statement.md — for Governance sign-off

Peer role: `ai-evaluation-engineer`, level 25 (Governance family). Target sign-off
date: 2026-11-19.

## Three residual risks

1. **Dropped tails not covered by any policy.** Traces head-sampled out AND not
   matching any tail rule are unrecoverable. If a novel failure class does not
   trip `keep_errors` / `keep_guardrail_trips` / `keep_slow`, we learn about it
   only when it shows up in the 5% base rate. Mitigation: a mod-107 "coverage-gap"
   scorer runs over the kept population and surfaces anomalous traces that would
   otherwise be at base rate only; promotion to the review store removes the
   information loss. Revisit if coverage-gap promotions exceed 2% of kept traffic
   — that would say we are keeping too narrow a tail set.

2. **Scrubbed content limits offline judges.** A groundedness judge reading reply
   content against retrieved documents will see `<email>` tokens where real text
   lived. Scores can be artificially low. Mitigation: judge-side handling —
   groundedness fetches a hash-correlated unscrubbed retrieval blob from the
   gated repro store (small, consented, access-audited). Revisit if the judge's
   false-negative rate on known-good examples exceeds 5%.

3. **Hashed users block cross-tenant analysis.** This is structural, not a bug.
   HMAC-SHA256 with per-tenant salt is by design non-joinable across tenants.
   Mitigation: cross-tenant analyses are limited to aggregated, non-user-identified
   metrics. Revisit only if a product requirement forces cross-tenant joins —
   that requires a separate, consented data-sharing agreement.

## What is escalated to Governance

- Retention windows (hot 14 / warm 60 / cold 180 days) — confirm compliance with
  GDPR Art. 5(1)(e) storage-limitation under our DPA.
- Cross-region export: confirm Langfuse's EU residency SKU is in place or route
  the SaaS egress to a non-cross-border endpoint.
- Consented-store scope: scope and access policy for the small gated unredacted
  store that mod-109 reviews feed from.
- Salt rotation policy: quarterly cadence approved; confirm KMS audit trail
  meets the Governance evidence bar.

Sign-off owner: `ai-evaluation-engineer` (level 25). Hand-off: this directory as
single-sitting artefact.
```

## 3. Validation steps

1. **`collector.yaml` validates**: `otelcol-contrib validate --config=collector.yaml` exits 0.
2. **One live trace lands in both backends after the Collector cold start**.
3. **Drill runs cleanly**: `python drill.py --runs 205` completes; 15 s wait allows the tail-buffer decision window.
4. **Policy columns in `drill-results.md` are populated from the backend** — a reviewer opens Phoenix and the exact count matches the "Observed keeps" column. Chapter 06: "do not trust that the config does what you wrote in `POLICY.md`; verify."
5. **PII counts are zero** in the backend after scrub for every seeded pattern. One documented near-miss with an explicit "add a dedicated regex" follow-up is acceptable; silent misses are not.
6. **SLI table uses observed drill numbers**, not expectations. The verdict column flips to `needs-a-targeted-tail-policy` on any row that fell short, with the policy change proposed inline.
7. **Residual-risk statement names three risks, mitigations, and the revisit trigger** — a Governance peer should be able to review in 15 minutes.
8. **No secrets committed** — `git grep -E '(sk-|pk-|Basic [A-Za-z0-9+/=]{20,})'` is empty.
9. **Ordering rule respected**: pipeline order is `attributes → … → tail_sampling → batch` with the rationale as a comment.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | `POLICY.md` covers every required section with concrete values. | §Surface / §Sampling / §Scrubbing / §Hashing / §Retention / §SLI / §Budget / §Residual / §Sign-off all filled. | Any section stubbed or "TBD" (except vendor unit prices). |
| R2 | `collector.yaml` validates. | `otelcol-contrib validate` exits 0. | Validation error. |
| R3 | Pipeline order `attributes → tail_sampling → batch` with rationale comment. | Order and rationale present. | Reordered or no rationale. |
| R4 | Every tail-sampling row in `drill-results.md` has `OK` or `investigate` with a one-line hypothesis. | All verdicts set and non-blank. | Missing verdicts. |
| R5 | Every seeded PII pattern shows zero (or documented tiny near-miss) in the backend. | Zero counts. | Silent PII leak. |
| R6 | `sli-preservation-check.md` uses observed drill numbers; policy change proposed inline if any row falls short. | Observed numbers present. | Still-expectation numbers only. |
| R7 | `residual-risk-statement.md` names three specific residuals, mitigations, and revisit triggers. | All three. | Fewer than three or trivial mitigations. |
| R8 | HMAC-SHA256 with per-tenant salt in KMS, citing GDPR Art. 4(5) and FIPS 198-1. | Both citations present. | Missing citation. |
| R9 | No secret in committed files. | `git grep` clean. | Any secret present. |
| R10 | Governance sign-off owner named (peer role `ai-evaluation-engineer`, level 25) with hand-off shape. | All three present. | Missing owner. |

A passing submission is 8+ of 10 with R1, R2, R3, R5, R8 all hard-required.

## 5. Common mistakes

- **Scrubbing *after* tail-sampling.** The tail buffer holds raw traces for 10 s by default; this is where raw PII would sit. Chapter 06 ordering rule exists for exactly this reason.
- **Copying chapter-06's YAML verbatim.** Chapter 06 says "cross-check names against the current README; do not copy." Policy names in `tailsamplingprocessor` have moved in past releases; `string_attribute` was previously called something else.
- **Setting head-sampling at the Collector only.** Head sampling at the Collector still forces the app to serialise and egress every trace; chapter 06 sets head at the SDK so the work does not happen.
- **Promising "no PII escapes."** Chapter 06: "do not claim in `POLICY.md` that the policy *guarantees* no PII escapes — say 'reduces to residual per drill results.'" Presidio is good, not perfect.
- **Budget math in abstract.** Chapter 06 wants arithmetic you can audit, even if $X and $Y are TBD. Writing "cost is bounded" is not sufficient.
- **SLI-preservation table with expectations only.** `sli-preservation-check.md` must use drill numbers; expectations live in `POLICY.md`.
- **Residual-risk statement with generic mitigations.** "We will be careful" is not a mitigation. Chapter 06 wants the structural answer (small consented store, judge-side handling, structural impossibility).
- **Unsalted SHA256.** Chapter 06 and chapter 04 both call this out; unsalted SHA256 of an email is a lookup-table attack.
- **Running the drill but not opening the backend.** Chapter 06: "read the backend" is the point. The config may do what the YAML says and still ship payload downstream if the regex doesn't match the key the application chose.
- **Not noting that Presidio is imperfect.** The residual-risk statement needs to name the miss rate and the mitigation.
- **Hot-store retention past 30 days** without a Governance sign-off rationale. GDPR Art. 5(1)(e) storage-limitation is the frame; a long hot window needs justification.

## 6. References

Official standards and project documentation:

- OpenTelemetry trace SDK — sampling primitives (`ParentBased`, `TraceIdRatioBased`, `AlwaysOn`, `AlwaysOff`), <https://opentelemetry.io/docs/specs/otel/trace/sdk/#sampling>.
- `opentelemetry-collector-contrib` — repo root, <https://github.com/open-telemetry/opentelemetry-collector-contrib>.
- `tailsamplingprocessor` (policy names change across releases; re-verify), <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/tailsamplingprocessor>.
- `attributesprocessor`, <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/attributesprocessor>.
- `redactionprocessor`, <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/redactionprocessor>.
- `probabilisticsamplerprocessor`, <https://github.com/open-telemetry/opentelemetry-collector-contrib/tree/main/processor/probabilisticsamplerprocessor>.
- Microsoft Presidio, <https://microsoft.github.io/presidio/>.
- NIST FIPS 198-1 — HMAC, <https://csrc.nist.gov/publications/detail/fips/198/1/final>.
- NIST SP 800-122 — PII handling, <https://csrc.nist.gov/publications/detail/sp/800-122/final>.
- GDPR — Regulation (EU) 2016/679, <https://eur-lex.europa.eu/eli/reg/2016/679/oj> (Art. 4(5) pseudonymisation, Art. 5(1)(e) storage limitation).
- OWASP Top 10 for Large Language Model Applications (LLM06 — Sensitive Information Disclosure), <https://owasp.org/www-project-top-10-for-large-language-model-applications/>.
- NIST AI Risk Management Framework (AI 100-1, 2023), <https://www.nist.gov/itl/ai-risk-management-framework> (this policy is a Measure-function artefact for mod-112).
- NIST AI RMF Generative AI Profile (AI 600-1), <https://airc.nist.gov/AI_RMF_Knowledge_Base/AI_RMF/Uses/GenAI-Profile>.

Local module chapters:

- `lessons/mod-102-trace-instrumentation/06-sampling-and-pii-policy.md` — head / tail / signal-driven sampling, scrubbing positions, Collector recipe, budget math, SLI-preservation check, residual-risk statement.
- `lessons/mod-102-trace-instrumentation/04-span-schema-design.md` — the schema whose attribute keys drive the scrub list and the tail-policy filters.
- `lessons/mod-102-trace-instrumentation/03-backend-bakeoff.md` — the Collector + two-backend fan-out that this policy builds on.
