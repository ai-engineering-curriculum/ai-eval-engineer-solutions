# exercise-05 — Product Side Model Card Slice — Solution

> Reference solution for `lessons/mod-112-owning-an-ai-eval-program/exercises/exercise-05-product-side-model-card-slice.md`.
> This is a **design** exercise. The deliverable is a product-side model card for the `support_agent` app family — a document that describes how the application is *used*, not how the underlying model was trained. Numbers, cohort names, and incident mentions in the worked card are **exercise fixture**; they illustrate the structure and cite only the running `support_agent` surface.

## 1. Solution overview

The "model card" originally described a trained model's capabilities and training data (as published by Mitchell et al. in 2019). The **product-side model card** is a different artefact: it describes an *application* that uses one or more models. It is written for the people who have to live with the application — a PM deciding what to launch, a Legal/Governance reviewer signing off on disclosure, a Safety lead preparing for an audit, a user-facing team writing release notes.

The product-side card is organised around *what the application does*, not *what the model is*:

1. **Who the application is for** (intended use, out-of-scope use, user cohorts).
2. **What it does** (features, surfaces, decision loops).
3. **How it performs** (quality, cost, latency per cohort, with CIs — not vanity point estimates).
4. **What it fails at** (known failure modes, representative incidents labelled as such, open issues).
5. **How it is governed** (release gates, review workflows, override policy, versioning).
6. **How it changes** (the specific pointer to the release-gate architecture and the delegation contracts so the card is not a one-time snapshot).

Five design choices anchor the solution:

1. **One card per app family, not per model.** The support_agent card references its current model pins, but the product does not change identity when the pin changes. A card that rotates on every model swap is a release note, not a card.
2. **Metrics with cohorts and CIs, never a single number.** `pass_rate = 0.82` is misleading; `pass_rate across cohorts: en 0.84 [0.82, 0.86] (n=1200), fr 0.73 [0.69, 0.77] (n=420) …` is a product card. NIST AI RMF `MEASURE 2.11` (fairness and bias) names this discipline.
3. **Known failure modes come from the incident regression set.** The failure section is anchored on exercise-04's fixtures — a failure mode either is in the regression set (and the card names which one) or it is not yet audited. "Fails occasionally on edge cases" is not a disclosure.
4. **Links, not duplication.** The card links out to release gate architecture (ex-01), delegation contracts (ex-02), and the regression set (ex-04). The card does not re-own what those documents own; it synthesises.
5. **The card lives with the product, not in a hidden folder.** Same repo as the product code; versioned alongside releases; the next revision has a known review date and owner.

```
                   product-side model card (support_agent)
                                  │
    ┌────────────────┬────────────┼────────────┬────────────────┐
    │                │            │            │                │
intended use   features     performance   known failures   governance
                                  │                            │
                                  ▼                            ▼
                     slice tables (cohort × metric)    release gate tree (ex-01)
                     with 95% CIs and n                delegation contracts (ex-02)
                                                       incident fixtures (ex-04)
```

## 2. Implementation — worked answer — `model-card-support-agent.md`

```markdown
# Product model card — support_agent
Version: 1.2  |  Date: 2026-11-19  |  Owner: support_agent eval lead
Next review: 2027-02-19 (quarterly) or on vendor-level model change.
Governing documents:
 - release gate architecture: `release-gate-architecture-support-agent.md`
 - delegation contracts: `delegation-contract-{safety,sre,ds,reviewops,legalgov,product}.md`
 - incident regression set: `incident_regression_v5.yaml`

## 1. Intended use

`support_agent` is a customer-support assistant integrated into the
e-commerce order-status and returns surface. It is intended to:

- Answer questions about a single user's **own orders** (status, shipping
  estimate, delivery exceptions).
- Explain **policy applicable to the user's order** (return window,
  final-sale exclusions, loyalty tier differences).
- Open a human-escalation ticket when the user is distressed, the
  question is out of scope, or policy requires a human.
- Draft (not send) notification text for the user; sending is a human
  action, gated by `mod-108` safety eval.

### 1.1 Out-of-scope uses

The application is **not** intended for:

- Legal, tax, or financial advice.
- Fraud decisions (approve / deny chargebacks, re-issue refunds without
  policy match). Fraud is a human + separate ML pipeline.
- Medical, product-safety, or recall queries. These escalate.
- Multi-user queries (an agent cannot look up another user's order).
- Account changes (address, payment method). These are gated.

Out-of-scope queries trigger the escalation path; see §3.3.

### 1.2 Intended users and cohorts

| Cohort | Audience | Supported locales | Notes |
|---|---|---|---|
| retail_consumer_en | Retail consumers | en-US, en-GB | Primary |
| retail_consumer_fr | Retail consumers | fr-FR | Secondary; lower volume |
| retail_consumer_ja | Retail consumers | ja-JP | Secondary; distinct policy wording |
| enterprise_buyer | B2B buyers | en-US | Smaller volume, higher stakes |

Service-tier dimensions tracked alongside cohort: `tenant_class ∈ {free, paid}`.

## 2. What the application does

### 2.1 Features and surfaces

| Surface | Decision loop | Tools used |
|---|---|---|
| `order_status_not_shipped_policy` | 1–2 turns; answer policy, offer escalation | docs.search, crm.lookup |
| `returns_in_window_refund` | 1–2 turns; confirm eligibility; open ticket | docs.search, crm.lookup, open_ticket |
| `escalate_to_human` | 1 turn; create ticket + user message | open_ticket |
| `faq_no_tools` | 1 turn; policy answer with cite | docs.search |

Trajectory contract (per mod-103 ex-01): `name` + `args` + `order` axes
plus `answer`, `groundedness`, `budget` axes with pass threshold 0.80.

### 2.2 Model pins and components

Current pins (as of this card version):

- Primary model: `(vendor)-main-2025-10-01` (support_agent brain).
- Judge for online eval: `(vendor)-judge-2025-10-01`, pinned prompt
  `judge_prompt_v3`.
- Retrieval index: `support_kb_2026_q4_sha256:ef34…`.
- Guardrail stack: `jailbreak_suite_support_agent_v7`, `pii_leak_suite_v4`.

Pin changes produce a new card version. Minor product-surface changes
(new intent, new tool) increment the card minor version; model-pin
changes may increment major. See §6.

### 2.3 Decision loop

1. Classify user intent.
2. Retrieve policy snippets (`docs.search`).
3. If a tool side effect is required (open_ticket, send_notification),
   compose argument and call.
4. Compose a grounded reply citing the retrieved snippet.
5. Score online: mod-107 ex-01 sampled trace scorer runs the quality
   judge; mod-108 safety scorer runs the safety judge.
6. If any gate trips post-reply, flag for human review (mod-109).

## 3. Performance

All numbers are **exercise fixture** for the purposes of this card
template. In production, these come from mod-111 ex-01's multi-objective
report at release time and from the online rollup afterwards.

### 3.1 Aggregate (last 7 days, prod)

| Metric | Point | 95% CI | n |
|---|---|---|---|
| Pass rate (quality judge) | 0.84 | [0.83, 0.85] | 18,420 |
| p95 latency | 1,840 ms | [1,780, 1,890] | 18,420 |
| p99 latency | 3,210 ms | — | 18,420 |
| Cost per 1k requests | $1.42 | [$1.38, $1.47] | 18,420 |
| Safety suite pass rate | 1.00 | [1.00, 1.00] (CI clipped) | 540 |

### 3.2 By cohort (last 7 days)

| Cohort | Pass rate (95% CI) | p95 latency ms | Cost $/1k | n |
|---|---|---|---|---|
| retail_consumer_en | 0.86 [0.85, 0.87] | 1,780 | 1.40 | 12,900 |
| retail_consumer_fr | 0.77 [0.74, 0.80] | 1,920 | 1.46 | 2,840 |
| retail_consumer_ja | 0.74 [0.69, 0.78] | 1,970 | 1.48 | 1,120 |
| enterprise_buyer | 0.82 [0.78, 0.85] | 2,050 | 1.52 | 1,560 |

The locale gap (en → fr/ja) is acknowledged; see §4 known limitations.

### 3.3 Escalation rates (last 7 days)

| Surface | Auto-escalation rate | Human-review trigger rate |
|---|---|---|
| `order_status_not_shipped_policy` | 4% | 6% |
| `returns_in_window_refund` | 7% | 9% |
| `escalate_to_human` | 100% (by design) | n/a |
| `faq_no_tools` | 1% | 2% |

Human-review trigger is sampled by mod-107 ex-01's sampler; the ratio
reflects the sampling policy, not an absolute quality signal.

## 4. Known limitations and failure modes

Each entry corresponds to a row in `incident_regression_v5.yaml`
(exercise-04) or an active ticket in the limitation register.

### 4.1 Final-sale refund promise (fixture FX-INC-2026-11-04-017)

The agent has previously promised a refund for a final-sale SKU when the
prompt lost a policy fragment during a length-reduction pass. **Current
state**: fixture row FX-INC-2026-11-04-017 is `block` in G11; the pre-fix
build is caught. **Residual risk**: SEV-2; demotion to `warn` eligible
after 2027-05-06 if six months green.

### 4.2 Lower performance in fr-FR and ja-JP cohorts

Pass rate in `retail_consumer_fr` and `retail_consumer_ja` is 7–12
points below the English cohort (§3.2). **Root cause (current best
understanding)**: the policy corpus has thinner coverage of locale-
specific return rules; retrieval returns the English policy when the
locale-specific one is missing. **Action**: content team is backfilling
localised policy docs (target: 2027-Q1). **Gate status**: G10 cohort
preservation currently `warn` (not `block`) because the delta is stable
over 90 days, not growing.

### 4.3 Multi-user queries

The agent cannot look up another user's order. Attempts fail with
`escalate_to_human`. **No fixture row** — this is by design and
not a regression candidate. Documented here for transparency.

### 4.4 Out-of-policy tool arguments

Programmatic argument matcher (mod-103 ex-01) catches most tool-argument
errors; a Level-4 (AST-based) matcher is roadmapped for the forthcoming
`sql_query_review` tool; not yet in production. **Gate status**: Level-2
(exact-value) matcher in G6; argument mismatch is a `block` at Level 2.

## 5. Safety and governance

### 5.1 Threat model

Primary concerns (all mapped to MITRE ATLAS and OWASP ML Top 10):

- **Prompt injection** (`AML.T0051.000`) — jailbreak suite G3 blocking.
- **Adversarial input** (`AML.T0043`) — regression fixtures G11 blocking.
- **PII leak** (`OWASP ML04` membership inference) — pii_leak_suite G3.
- **Supply chain risk** (`OWASP ML06`) — vendor model pinned; pin change
  requires DS + Safety PR review.
- **Data poisoning on retrieval index** (`OWASP ML02`) — the retrieval
  index is content-moderated upstream; `docs.search` tool is read-only.

### 5.2 Guardrails

| Guardrail | Mechanism | Gate it feeds |
|---|---|---|
| Jailbreak suite | Published red-team prompts, pinned version | G3 (CI), G8 (online quality) |
| PII leak suite | Templated prompts that attempt PII extraction | G3 (CI) |
| Unsafe-tool suite | Tool sequences that violate policy (e.g. open_ticket without lookup) | G3 (CI), G6 (trajectory) |
| Safety judge (online) | mod-108 safety judge, pinned | G8 (continuous) |

### 5.3 Human review

Human review runs on a sampled set (mod-107 ex-01 sampler) with IAA
tracked (mod-109 ex-03). The current reviewer pool IAA floor is 0.70
(Cohen's κ, helpfulness axis). Review Ops owns pool, training, and
disagreement triage per `delegation-contract-reviewops.md`.

### 5.4 Overrides

Override policy is in `release-gate-architecture-support-agent.md` §7.
Safety and incident-gate overrides require two signers, time-limited.
Overrides are logged and visible in the release history view.

### 5.5 Data handling

- Traces: retained 90 days online, 1 year cold. PII redacted at ingest
  by OTel processor; review traces redacted again at human-review export.
- Eval traces for regression fixtures: PII redacted per the fixture
  redaction manifest (exercise-04 §2.3).
- Human-review decisions are retained but reviewer identities are
  pseudonymised in the gate audit log.

Residency and retention policy is owned by Legal/Governance per
`delegation-contract-legalgov.md`.

## 6. Versioning

The card is versioned with the app family, not the model:

- **Minor bump (1.x → 1.x+1)**: cohort gains/losses, performance table
  refresh, new fixture row, new surface, new limitation.
- **Major bump (1.x → 2.0)**: primary model-pin change; new tool with a
  side effect; a change that invalidates the §1 intended use.

Each bump reserves a line in a changelog section at the end of the card.
The card is PR-reviewed with the release that motivates it; a card
change that is not tied to a release is a documentation PR with
`support_agent eval lead` + Legal reviewer.

## 7. Accessibility and user disclosure

Users are shown an AI-authored notice on each reply ("An AI assistant
drafted this — a human is available on request"). The exact copy is
owned by Product and Legal per `delegation-contract-product.md` and
`delegation-contract-legalgov.md`. A card change does not change the
disclosure copy; a disclosure-copy change does not change the card.

## 8. How to raise an issue

- **Correctness**: open ticket in the eval lead's rotation; link the
  trace ID.
- **Safety**: page the safety on-call; the issue becomes a candidate
  for a fixture row (exercise-04).
- **Governance**: contact Legal/Governance per their delegation
  contract; disclosures and regulator queries route here.

## 9. Changelog

| Version | Date | Change |
|---|---|---|
| 1.2 | 2026-11-19 | Added fixture reference for INC-2026-11-04-017; updated §3 numbers; refreshed cohort table. |
| 1.1 | 2026-10-08 | Added ja-JP cohort; updated threat model with ATLAS AML.T0043 reference. |
| 1.0 | 2026-09-15 | Initial product card for support_agent. |

## 10. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework>.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/>.
- MITRE ATLAS, <https://atlas.mitre.org/>.
- Mitchell et al., "Model Cards for Model Reporting" (2019), <https://arxiv.org/abs/1810.03993> — the original model card concept; this product card is a product-family specialisation of it.
```

## 3. Validation steps

A reviewer validates the submission by:

1. **Open the card and read §1.** Intended use is specific to a product surface; out-of-scope uses are named; cohorts are listed. A card that starts with "this is a general-purpose assistant" is a different artefact.
2. **Open §3 performance tables.** Every metric has a point, a 95% CI, and an `n`. A metric without a CI fails; a metric without an `n` fails; a metric without a cohort breakdown fails for product cards.
3. **Open §4 limitations.** Every entry maps to an incident fixture (exercise-04) or an active ticket. "Fails on edge cases" without specificity fails; a limitation that has been incidented but has no fixture row is listed as open with an owner.
4. **Walk §5 threat model.** Threats map to MITRE ATLAS techniques and OWASP ML Top 10 entries. A card that says "secure by design" without named threats is not a threat model.
5. **Walk the links.** The card points to release gate architecture, delegation contracts, and the regression set. Each link resolves.
6. **Walk the override story.** The card references the override policy from exercise-01; it does not re-invent one. A card that writes a new override policy is drifting from the release doc.
7. **Walk §6 versioning.** Minor vs major bump is defined; the card is versioned with the app family, not the model pin. A card that changes version on every model pin is a release note in disguise.
8. **Walk §7 disclosure.** User-facing disclosure copy is owned by named teams per delegation contract. A card that writes customer-facing text is overstepping Legal/Gov scope.
9. **Walk §9 changelog.** The changelog entries have a date, a version, and a change summary. The card is reviewable as it ages.
10. **Human readthrough with each named audience.** A PM, a Legal reviewer, a Safety lead, and a new engineer each read the card and can answer their question without asking around. If any audience needs ten follow-up questions, the card is written for the wrong audience.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Card is organised around the **application**, not the model's training data. | §1 names product surfaces and cohorts. | Reads like a trained-model card (training data, architecture). |
| R2 | §1 lists intended use, out-of-scope use, and user cohorts. | All three present and specific. | Missing out-of-scope section. |
| R3 | §3 performance tables carry CI and n per metric; cohort breakdown present. | CI + n + cohort columns. | Single point estimate for aggregate only. |
| R4 | §4 limitations map to incident fixtures (exercise-04) or named open tickets. | Each limitation has a fixture_id or ticket_id. | "Fails on edge cases" generic text. |
| R5 | §5 threat model names MITRE ATLAS / OWASP ML entries and maps each to a guardrail. | Named techniques / entries; mapped guardrails. | "Secure by design" without specifics. |
| R6 | Card references release gate architecture (ex-01), delegation contracts (ex-02), and regression set (ex-04) by file name. | All three links present. | Card re-owns gate rules, override policy, or incident list. |
| R7 | §6 versioning defines minor vs major bump; major tied to intended-use-affecting changes. | Rules present; aligned with product release cadence. | Version churns on every model pin. |
| R8 | §7 disclosure is co-owned with Product and Legal per delegation contracts (does not duplicate copy). | Ownership named; no customer-facing copy in the card. | Card writes marketing text or disclosure copy. |
| R9 | §9 changelog present with dates and versions. | Table + at least 2 entries. | No changelog. |
| R10 | Named standards cited in §10 with URLs: NIST AI RMF, OWASP ML Top 10, MITRE ATLAS, and Mitchell et al. 2019 for the model-card concept. | All four cited. | Standards named without citation. |

A passing submission is 8+ of 10 with **R1, R3, R4, R5, R6** hard-required.

## 5. Common mistakes

- **Writing a trained-model card.** A product card does not re-list training data, architecture layers, or compute budget. It lists product surfaces, cohorts, decision loops, and gates.
- **Point-estimate metrics with no cohorts.** `pass_rate = 0.82` hides that fr-FR is 0.73. A product card without cohort breakdown is misleading, and NIST AI RMF `MEASURE 2.11` is the reason.
- **"Fails occasionally on edge cases."** An unspecific limitation protects no one. Each limitation has a named root cause and either a fixture row or an open ticket.
- **Rewriting the override policy inside the card.** The override policy belongs to the release gate architecture (exercise-01). The card references it. Duplication drifts; one of the two will be wrong six months from now.
- **Versioning the card on every model pin.** Pins change weekly; products change quarterly. Version the card with the product; carry pin history in §2.2.
- **No threat model.** A card without a threat section is a brochure. Even if the application has not been attacked yet, the techniques it must defend against are the threat model.
- **Omitting data-handling.** Retention, residency, and pseudonymisation are first-class for the audiences the card is written for. "Standard practices apply" is not a disclosure.
- **Customer-facing copy in the card.** The card is an internal governance document. A card that drafts user-facing text belongs to Product + Legal, not to the eval lead's weekend.
- **Missing links to the governing docs.** A card without links is isolated — the next reader will not find the gate tree or the delegation contracts and will reinvent them inconsistently.
- **No review date or review owner.** Cards are living documents. A card without a next-review date is a snapshot that will age out quietly.
- **Performance table from the demo, not prod.** The numbers in §3 come from the production rollup, not a cherry-picked eval run. Cite the mod-111 report for the release-time numbers; cite the online rollup for the ongoing numbers.
- **Treating the card as a compliance deliverable.** Compliance reads the card; compliance is not the only reader. The card exists so the PM, the Safety lead, and the next eval lead can make a call without a meeting.

## 6. References

Official standards and foundational references:

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework>. The product card is organised against `MAP 1.1` (intended purpose), `MAP 2.3` (stakeholder mapping), `MEASURE 2.11` (bias/fairness), `MEASURE 4.2` (qualitative + quantitative analysis), `MANAGE 4.3` (post-deployment public reporting). Every section of the card maps to one or more of these.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/>. §5 threat model cites `ML02` (data poisoning), `ML04` (membership inference), `ML05` (model theft), `ML06` (supply chain).
- MITRE ATLAS, <https://atlas.mitre.org/>. §5 threat model cites `AML.T0051.000` (LLM prompt injection), `AML.T0043` (craft adversarial input). The framework's case-study catalogue informs the structure of §4 known limitations.
- Mitchell, M., et al., "Model Cards for Model Reporting", FAT* 2019, <https://arxiv.org/abs/1810.03993>. This paper introduced the model-card concept. The product-side card is a product-family specialisation; the "quantitative analyses" and "intended use" sections descend directly from Mitchell et al.

Local solutions this card reads from:

- `mod-112-owning-an-ai-eval-program/exercise-01` (release gate architecture) — governs §5.4 overrides.
- `mod-112-owning-an-ai-eval-program/exercise-02` (delegation contracts) — governs §5.3, §5.5, §7, §8.
- `mod-112-owning-an-ai-eval-program/exercise-04` (incident regression set) — governs §4.
- `mod-111-cost-latency-quality-tradeoff/exercise-01` (multi-objective report) — provides §3 release-time numbers.
- `mod-107-online-eval-and-regression/exercise-01` (sampled online eval) — provides §3 ongoing numbers.
- `mod-109-human-review-workflows/exercise-03` (IAA) — provides §5.3 reviewer-pool floor.
- `mod-110-eval-data-platform-slice/exercise-01` (schema + lineage) — provides pin IDs cited in §2.2.

<!-- All numbers in §3, cohort names in §1.2, incident references in §4, and pins in §2.2 are exercise fixture tied to the running support_agent surface. They illustrate the card structure; do not cite as vendor benchmarks. -->
