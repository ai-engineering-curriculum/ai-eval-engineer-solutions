# exercise-02 — Delegation Contracts With Peer Specialists — Solution

> Reference solution for `lessons/mod-112-owning-an-ai-eval-program/exercises/exercise-02-delegation-contracts-with-peer-specialists.md`.
> This is a **design** exercise. The deliverable is a set of written contracts between the eval lead and each peer specialist (Safety, SRE, Data Science, Review Ops, Legal/Governance, Product) that names who owns what, who answers questions about what, and how handoffs happen when a gate fires. Worked contracts below use the `support_agent` surface and are **exercise fixture** — any named person, team, or SLO is illustrative, not a vendor claim.

## 1. Solution overview

A delegation contract is the single-page answer to *"when the eval gate fires at 3 a.m., who gets paged and what are they expected to do?"* The eval lead cannot own every gate, every dataset, every rubric, every dashboard — and should not. The contract is how that is said out loud, in writing, before an incident.

Each contract is a short, readable document with six load-bearing sections:

1. **Scope.** The peer's area of ownership, named in terms of *decisions the eval lead will not make without the peer*. Scope is written to be falsifiable — a reviewer can point at a question and say "yes, that's in scope" or "no, that's the eval lead's".
2. **RACI (five rows).** For each of five canonical artefacts (dataset, rubric/metric, gate rule, dashboard, incident), name who is Responsible, Accountable, Consulted, Informed. No more than one Accountable per row.
3. **SLAs.** The specialist's response time, by severity, for the two directions the handoff runs: *eval lead asking the peer* and *peer asking the eval lead*. Business hours vs off-hours spelled out.
4. **Escalation ladder.** Named rotation first, then a named role, then the on-call manager. No named individuals (people change jobs).
5. **Explicit non-coverage.** What the peer *will not* do. A contract that says "Safety covers everything safety-shaped" leaves ambiguity for the gate that needs to be re-run; a contract that says "Safety does not re-run offline eval; the eval lead does" closes it.
6. **Review cadence.** When the contract is revisited, by whom, and what triggers an out-of-cycle review (a new product launch, a SEV-1 incident, a major model vendor change).

Five design choices anchor every contract:

1. **RACI rows are keyed on *artefacts*, not *activities*.** "Who owns the safety dataset" is auditable; "who does safety work" is not. The five artefacts (dataset, rubric, gate rule, dashboard, incident) are the ones from mod-110 ex-01 — the same objects already in the platform.
2. **One Accountable per artefact.** A row with two A's is the design bug mod-112 chapter 03 calls "shared Accountable is no Accountable". The eval lead's job is to make the A unambiguous, even when it means owning less.
3. **SLAs are two-way.** The specialist can ask the eval lead for a dataset refresh or a rubric change. The contract SLAs cover both directions; a one-way SLA is a demand, not a contract.
4. **Non-coverage is explicit.** The hardest part of the contract is the "we do not do this" bullets. Without them, a gate failure sits in a Slack channel for a day waiting for someone to pick it up.
5. **Written against named NIST AI RMF `GOVERN` functions.** `GOVERN 1.2` (accountability roles), `GOVERN 2.1` (role clarity), and `GOVERN 4.1` (incident reporting) are the functions the contract is organised around; a reviewer can trace each section back to one.

```
                            eval lead
                                │
          ┌─────────┬───────────┼───────────┬───────────┬─────────┐
          │         │           │           │           │         │
        Safety     SRE     Data Science  Review Ops  Legal/Gov  Product
          │         │           │           │           │         │
      G3, parts  G9, parts  rubric/score  G7, review  model card  SLO
      of G11     of SLO     authoring     ops, IAA    + comms     + weights
```

The eval lead is the *integrator* — the shared verdict schema, the gate tree, and the release policy are the eval lead's. Each peer owns a slice.

## 2. Implementation — worked answer — contract set for `support_agent`

### 2.1 `delegation-contract-safety.md`

```markdown
# Delegation contract — Safety team × Eval Lead
App family: support_agent  |  Version: 1.0  |  Effective: 2026-11-19
Review cadence: quarterly (next: 2027-02-15)

## 1. Scope

Safety owns the **jailbreak**, **PII leak**, and **unsafe-tool-use** test
suites and their verdicts for the `support_agent` surface. The eval lead
owns everything else — quality rubrics, cost / latency gates, canary
runbook, dashboard layout, release policy file.

## 2. RACI

| Artefact | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| `jailbreak_suite_support_agent_v<N>` dataset | Safety | Safety lead | Eval lead | Product |
| Safety rubric (binary unsafe flag + reason taxonomy) | Safety | Safety lead | Eval lead | Review Ops |
| Gate G3 (CI safety gate) rule | Eval lead | Eval lead | Safety lead | SRE |
| Safety dashboard (pass rate × suite × week) | Safety | Safety lead | Eval lead | Product, Legal |
| Safety incident (G3 fail in prod, SEV-1) | Safety on-call | Safety lead | Eval lead, SRE | Product, Legal |

One Accountable per row. Gate rule is the eval lead's because the gate
tree lives in release policy, not in the safety suite.

## 3. SLAs

### Eval lead → Safety

| Request | Business hours | Off-hours |
|---|---|---|
| "Is this prompt change safety-affecting?" | 1 business day | n/a (not off-hours) |
| "Refresh suite for new jailbreak pattern we saw in prod" | 3 business days for stub, 2 weeks for full suite | n/a |
| "Safety sign-off on release X" | 1 business day | n/a |
| "Safety gate fired on canary, what do we do?" | 30 min ack, 4 h path forward | 1 h ack (SEV-1), 4 h path forward |

### Safety → Eval lead

| Request | Business hours | Off-hours |
|---|---|---|
| "New jailbreak suite version is ready — please bump G3 pin" | 1 business day | n/a |
| "Prompt change in `support_agent` looks safety-risky — please block" | 15 min ack if SEV-1, 1 h ack otherwise | 1 h ack (SEV-1) |
| "We need a dataset refresh to reproduce an incident" | 2 business days | n/a |

## 4. Escalation ladder

For a G3 failure in prod:

1. `safety-oncall` (rotation, not a person)
2. Safety lead role
3. VP Safety / VP Engineering (joint page)

A SEV-1 that is unacked past the SLO escalates automatically via the
incident tool; no human needs to walk the ladder manually.

## 5. Explicit non-coverage

Safety will **not**:

- Author quality rubrics (helpfulness, groundedness) — Data Science owns.
- Re-run the offline quality eval (G2) — eval lead owns.
- Author the model card (ex-05) — eval lead + Legal co-own.
- Approve a safety-gate override by themselves — override requires
  *both* Safety lead and Eval lead signatures (per release gate ex-01 §7).
- Own the retrieval index content moderation — the Content team owns that
  layer upstream of the agent.

Eval lead will **not**:

- Edit a safety suite's test cases without a Safety PR review.
- Rewrite the safety reason taxonomy — Safety owns the vocabulary.
- Override a G3 fail solo.

## 6. Review

- Quarterly review by Eval lead + Safety lead, aligned with the
  release-policy review cadence.
- Out-of-cycle review triggered by: a SEV-1 safety incident; a model
  vendor change; a new MITRE ATLAS technique the team wants to defend
  against.
- The contract owner-of-record is the Eval lead; Safety lead co-signs.

## 7. Governance mapping

- NIST AI RMF `GOVERN 1.2` (accountability roles) — §2 RACI.
- NIST AI RMF `GOVERN 2.1` (role clarity, training) — §5 non-coverage.
- NIST AI RMF `GOVERN 4.1` (incident reporting) — §4 escalation + §3 SLA.
- MITRE ATLAS `AML.T0051.000` (LLM prompt injection), `AML.T0043` (craft
  adversarial input) — jailbreak suite coverage; new techniques trigger
  out-of-cycle review.
```

### 2.2 `delegation-contract-sre.md` (abbreviated form; full table structure identical to 2.1)

```markdown
# Delegation contract — SRE × Eval Lead

## 1. Scope

SRE owns the **latency SLO** and the **availability** of the eval runner
infrastructure. Eval lead owns every quality / safety / cost gate rule
and the release policy file.

## 2. RACI

| Artefact | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| Latency SLO table (p50/p95/p99 per tenant class) | SRE | SRE lead | Eval lead, Product | Finance |
| Eval runner availability / throughput | SRE | SRE lead | Eval lead | Eval platform |
| Gate G9 (canary latency SLO) rule | Eval lead | Eval lead | SRE lead | — |
| Latency dashboard (prod + canary) | SRE | SRE lead | Eval lead | Product |
| Latency incident (p95 over SLO in prod, SEV-1) | SRE on-call | SRE lead | Eval lead, Product | Finance |

## 3. SLAs (headline rows)

- Eval lead → SRE: "Canary p95 CI floor is over SLO, is this real?" — 15
  min ack during canary, 1 h path forward.
- SRE → Eval lead: "We need to raise p95 SLO because of an infra change" —
  1 business day review, 3 business days approval (requires Product sign).

## 4. Escalation ladder

`sre-oncall` → SRE lead → VP Engineering.

## 5. Explicit non-coverage

SRE will **not**:

- Approve a quality-gate override.
- Own the cost gate (G4) — eval lead owns, Finance consulted.
- Author rubrics or datasets.

Eval lead will **not**:

- Raise or lower the latency SLO without SRE review.
- Add a new gate that reads infra signals (e.g. GPU utilisation) without
  SRE co-signing — infra signals have false-fail profiles the eval lead
  does not have the context to tune.

## 6. Review

Quarterly; out-of-cycle on infra topology change (new region, new
inference stack) or on a SEV-1 latency incident.

## 7. Governance mapping

- NIST AI RMF `GOVERN 1.2`, `MEASURE 2.6` (computational infrastructure),
  `MANAGE 2.3` (continuous monitoring).
```

### 2.3 `delegation-contract-data-science.md`

```markdown
# Delegation contract — Data Science × Eval Lead

## 1. Scope

Data Science owns the **quality rubric** vocabulary (what "helpful",
"grounded", "concise" mean) and the **judge model choice + calibration**.
Eval lead owns the gate rules that read those rubric scores and the
release policy.

## 2. RACI

| Artefact | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| Rubric definition (helpfulness, groundedness, …) | Data Science | DS lead | Eval lead, Product | Review Ops |
| Judge model choice (model + prompt + temperature) | Data Science | DS lead | Eval lead, Safety | — |
| Judge calibration set | Data Science | DS lead | Eval lead, Review Ops | — |
| Gate G2 (offline paired quality) rule | Eval lead | Eval lead | DS lead | — |
| Quality dashboard | Data Science | DS lead | Eval lead | Product |

## 3. SLAs

- Eval lead → DS: "New product surface — need rubric by Friday" — 1
  business week for v0, 1 business month for calibrated v1.
- DS → Eval lead: "Judge model vendor announced deprecation — need gate
  pin bump" — treated as SEV-2 by default; eval lead acks in 1 business
  day; policy PR within 2 business weeks.

## 4. Escalation ladder

`data-science-oncall` (if present) → DS lead → VP Data / VP Engineering.
If DS has no on-call rotation, DS lead is the first page during business
hours; off-hours quality questions escalate to eval lead on-call directly.

## 5. Explicit non-coverage

Data Science will **not**:

- Own the production gate rule — "quality ≥ 0.8" as a threshold decision
  is the eval lead's; "helpfulness is a 0–1 scalar" as a measurement
  decision is DS's.
- Approve safety or cost overrides.
- Own the human-review workflow — Review Ops owns.

Eval lead will **not**:

- Change the rubric vocabulary.
- Change the judge model, prompt, or temperature without a DS PR review.
- Publish a quality dashboard with a metric Data Science has not defined.

## 6. Review

Quarterly. Out-of-cycle on a new app family, a major model vendor change,
or a drift alert from mod-107 ex-02.

## 7. Governance mapping

- NIST AI RMF `MEASURE 2.7` (measurement validity), `MEASURE 2.11`
  (fairness), `GOVERN 1.2`.
- OWASP ML Top 10 `ML07` (transfer-learning attacks) — when DS swaps the
  judge model, the eval lead reruns the calibration gate.
```

### 2.4 `delegation-contract-review-ops.md`

```markdown
# Delegation contract — Review Ops × Eval Lead

## 1. Scope

Review Ops owns **human review workflows** — reviewer onboarding,
reviewer training, IAA tracking, disagreement triage, reviewer payment.
Eval lead owns the gate rule that reads the review outcomes (G7) and
the sampling policy that decides which traces go to review.

## 2. RACI

| Artefact | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| Reviewer pool (hiring, training, offboarding) | Review Ops | Review Ops lead | Eval lead | Finance |
| IAA calibration set | Review Ops | Review Ops lead | Eval lead, DS | — |
| Disagreement triage policy | Review Ops | Review Ops lead | Eval lead | Safety (for safety-flag disagreements) |
| Gate G7 (pre-prod human review) rule | Eval lead | Eval lead | Review Ops lead | — |
| Sampling policy (which traces go to review) | Eval lead | Eval lead | Review Ops lead, Legal | — |

## 3. SLAs

- Eval lead → Review Ops: "50 traces for the pre-prod bake, needed in 24 h"
  — 1 business day turnaround at normal cadence; 4 h at SEV-1.
- Review Ops → Eval lead: "IAA has dropped below floor on helpfulness —
  rubric needs a refresh" — 1 business day ack; eval lead opens a DS
  rubric-change PR within 1 business week.

## 4. Escalation ladder

`review-ops-oncall` (contracted vendor paging) → Review Ops lead →
VP Operations / VP Engineering.

## 5. Explicit non-coverage

Review Ops will **not**:

- Author rubrics (DS owns).
- Approve production safety overrides.
- Own the public leaderboard / metrics page.

Eval lead will **not**:

- Contact reviewers directly.
- Change the disagreement triage policy.
- Publish a reviewer-facing metric (reviewer-level pass rate, reviewer
  throughput) — those metrics are HR-sensitive and Review Ops owns them.

## 6. Review

Quarterly. Out-of-cycle on a vendor change (new reviewer provider), a
privacy-law change (reviewer location, data residency), or an IAA floor
breach.

## 7. Governance mapping

- NIST AI RMF `GOVERN 3.1` (human roles in AI systems), `GOVERN 3.2`
  (training), `MEASURE 2.10` (human-AI configuration).
```

### 2.5 `delegation-contract-legal-governance.md`

```markdown
# Delegation contract — Legal/Governance × Eval Lead

## 1. Scope

Legal/Governance owns the **model card** external-facing text, the
**privacy / data-residency policy** for eval traces, and the **regulator
disclosure** workflow. Eval lead owns the internal eval metrics the model
card reads from and the retention of underlying traces.

## 2. RACI

| Artefact | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| Model card external text (ex-05) | Legal/Gov | Legal lead | Eval lead, Product, Safety | Review Ops |
| Data retention / residency policy for traces | Legal/Gov | Legal lead | Eval lead, SRE, Review Ops | Security |
| Regulator-facing disclosure doc | Legal/Gov | Legal lead | Eval lead, Safety | — |
| Internal eval metrics feeding the model card | Eval lead | Eval lead | Legal, DS | Product |
| Incident disclosure timing for a SEV-1 | Legal/Gov | Legal lead | Eval lead, Safety | VP Engineering |

## 3. SLAs

- Eval lead → Legal: "Need model card v1.1 for product launch in 4 weeks"
  — 2 business weeks to review, 1 business week to publish.
- Legal → Eval lead: "New regulator query — need latency, safety, fairness
  rollup by Thursday" — 2 business days for the data pull; longer-running
  queries converted into a dashboard rather than a one-off.

## 4. Escalation ladder

Legal counsel → Legal lead → General Counsel.

## 5. Explicit non-coverage

Legal/Governance will **not**:

- Author internal metric definitions.
- Approve production safety overrides.
- Author or edit prompts / tools / datasets.

Eval lead will **not**:

- Write customer-facing model-card text.
- Approve external disclosure timing.
- Change trace retention policy without a Legal sign.

## 6. Review

Quarterly. Out-of-cycle on new regulation (e.g. jurisdictional AI act
passing), a SEV-1 incident with disclosure obligation, or a change to
data processors.

## 7. Governance mapping

- NIST AI RMF `GOVERN 1.1` (legal/regulatory requirements), `GOVERN 4.1`
  (incident reporting), `MANAGE 4.3` (post-deployment monitoring + public
  reporting).
- OWASP ML Top 10 `ML03` (model inversion) / `ML04` (membership
  inference) — residency and retention of eval traces shape the attack
  surface here.
```

### 2.6 `delegation-contract-product.md`

```markdown
# Delegation contract — Product × Eval Lead

## 1. Scope

Product owns the **product SLO** (what "good" means for the user), the
**weights** on the multi-objective score (quality vs cost vs latency),
and the **launch calendar**. Eval lead owns the measurement and the
gate rules; the eval lead does not set weights.

## 2. RACI

| Artefact | Responsible | Accountable | Consulted | Informed |
|---|---|---|---|---|
| Product SLO (what "good" means) | Product | PM | Eval lead, DS, SRE | Safety |
| Multi-objective weights (mod-111 ex-01) | Product | PM | Eval lead, Finance | DS |
| Launch calendar | Product | PM | Eval lead, Safety, SRE | — |
| Internal rollout dashboard | Eval lead | Eval lead | PM | SRE |
| User-facing status page (quality / availability) | Product (copy), SRE (data) | PM | Eval lead, Legal | — |

## 3. SLAs

- Eval lead → Product: "Weight change request based on a cost-regression
  canary" — 1 business week review; the weight change requires Finance
  sign if cost weight moves.
- Product → Eval lead: "Launch date moved up by two weeks — can we still
  pass the gates" — 1 business day feasibility; the eval lead does not
  compress gate SLOs to fit a launch date.

## 4. Escalation ladder

PM → Product lead → VP Product / VP Engineering.

## 5. Explicit non-coverage

Product will **not**:

- Override a safety or incident-regression gate.
- Author the rubric or the judge.
- Set latency or cost thresholds unilaterally — PM sets *targets*, SRE
  and Finance co-set the *thresholds* the gates read.

Eval lead will **not**:

- Set the multi-objective weights.
- Publish a user-facing quality metric without PM sign.
- Change the launch calendar.

## 6. Review

Quarterly, aligned with the product planning cycle. Out-of-cycle on a
new product surface or a material SLO change.

## 7. Governance mapping

- NIST AI RMF `MAP 1.1` (intended purpose), `MAP 2.3` (mapping to
  stakeholders), `MANAGE 1.3` (risk/benefit trade-offs).
```

## 3. Validation steps

A reviewer validates the submission by:

1. **Count the contracts.** Six specialists are named in §1 overview (Safety, SRE, Data Science, Review Ops, Legal/Governance, Product). A submission that stops at two or three is incomplete.
2. **Open any contract and check the six sections.** Scope, RACI, SLA, escalation, non-coverage, review cadence. A contract missing any section fails.
3. **Trace an Accountable.** Pick any artefact (e.g. "jailbreak suite"). It appears in exactly one contract's RACI, with exactly one Accountable. If a peer's non-coverage says "we do not do X", then X appears in another peer's Accountable column — close the loop.
4. **Walk two handoffs.** Pick a safety incident; walk from "gate G3 fired" to resolution using only the contracts and the gate tree. Repeat for a latency incident. If either walk hits "we would figure it out", the contracts are incomplete.
5. **Check two-way SLAs.** Every contract names both "eval lead → specialist" and "specialist → eval lead" response times. One-way SLAs are a demand, not a contract.
6. **Check the escalation ladder has no named individuals.** People change jobs; roles do not. A ladder that says "Alice first, then Bob" is one departure away from obsolete.
7. **Check non-coverage.** Each contract has at least three "will not" bullets — one for the peer, two for the eval lead (or inverse). "We cover everything about X" is not non-coverage.
8. **Check governance mapping.** Every contract cites specific NIST AI RMF functions and (where relevant) OWASP ML Top 10 entries. "Aligned with best practice" without a function name is a slogan.
9. **Check review cadence.** Every contract names a next review date and at least one out-of-cycle trigger. A contract without a review cadence is a fossil.
10. **Human readthrough.** A new eval lead picks up all six contracts and can answer "who do I page for what" without asking around. The contracts are written for the eval lead's successor, not for the current eval lead.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Six contracts delivered (Safety, SRE, DS, Review Ops, Legal/Gov, Product). | All six present. | Two or three contracts only. |
| R2 | Each contract has six sections: scope, RACI, SLA, escalation, non-coverage, review. | All six sections per contract. | A contract missing a section. |
| R3 | RACI keyed on artefacts (dataset / rubric / gate / dashboard / incident), not activities. | Artefact-keyed rows. | Activity-keyed rows ("safety work"). |
| R4 | Exactly one Accountable per artefact. | One A per row. | Shared A ("safety lead and eval lead"). |
| R5 | SLAs are two-way, by severity, business vs off-hours. | Both directions, both severities. | One-way SLAs; no severity breakdown. |
| R6 | Escalation ladder uses rotations and roles, not individuals. | Pager groups / role titles. | Named individuals. |
| R7 | Non-coverage is explicit on both sides (what the peer will not do, what the eval lead will not do). | Both directions, ≥3 bullets per side. | One-sided non-coverage; "we do everything". |
| R8 | Every artefact in any peer's non-coverage appears in another peer's RACI as Accountable. | Closed loop. | Orphaned artefacts. |
| R9 | Each contract maps to specific NIST AI RMF functions (and other standards where relevant). | Function IDs named. | "Aligned with NIST" without citation. |
| R10 | Review cadence named; out-of-cycle triggers named. | Both named. | "We'll review it sometime." |

A passing submission is 8+ of 10 with **R3, R4, R6, R7, R8** hard-required.

## 5. Common mistakes

- **One big contract.** A single document covering every peer looks tidy but nobody reads it. Each peer owns their own contract; the eval lead signs all six.
- **Shared Accountable.** "Safety lead and eval lead are jointly accountable for the safety suite." In practice that means neither is, and the gate failure sits unowned. One A per row; if coordination is needed, use Consulted.
- **RACI rows on activities instead of artefacts.** "Who does safety work" is an argument; "who owns `jailbreak_suite_support_agent_v7.yaml`" is an audit.
- **Omitting non-coverage.** Without explicit "we do not do X" bullets, every gate failure becomes a jurisdictional dispute. Write the awkward bullets; they prevent the awkward Slack threads.
- **One-way SLAs.** A contract that only says what the specialist owes the eval lead is a demand. The eval lead's obligations (dataset refresh turnaround, rubric change PR speed) go in the same table.
- **Named individuals in escalation.** Every roster change invalidates the contract. Use rotations and roles; let the incident tool resolve the current human.
- **No off-hours story.** Every gate that can fire in prod can fire at 3 a.m. A contract without an off-hours row is a contract for business-hours incidents only.
- **Review cadence missing.** Contracts age. Every six months a vendor changes, a regulation passes, a team reorganises. A contract without a review date is a contract that doesn't update.
- **Governance mapping as a slogan.** "Covers NIST AI RMF" without a function ID is unverifiable. Name `GOVERN 1.2`, `GOVERN 2.1`, `GOVERN 3.1` — the reviewer can trace to the published framework.
- **Conflating product SLO and gate threshold.** Product sets the *target* ("p95 ≤ 2.5 s is good"); SRE + Finance set the *threshold the gate reads* ("block if p95 CI floor > 3.0 s"). They are related but not the same.
- **Legal owns the data; eval lead owns the data.** Pick one. In this solution, Legal/Governance owns the retention and residency *policy*; Eval lead owns the operational retention. The RACI makes the split explicit.
- **Treating the eval lead as a gate owner for every gate.** The eval lead is the integrator — the schema, the gate tree, the release policy are the eval lead's. The *gate rules* are co-owned with the relevant peer. A contract that makes the eval lead responsible for every gate rule in isolation is where burnout comes from.

## 6. References

Official standards:

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework>. Contracts are structured around `GOVERN 1.1` (legal/regulatory), `GOVERN 1.2` (accountability roles), `GOVERN 2.1` (role clarity), `GOVERN 3.1` (human roles in AI systems), `GOVERN 3.2` (training), `GOVERN 4.1` (incident reporting). Individual contracts cite additional `MAP`, `MEASURE`, `MANAGE` functions where relevant.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/>. `ML03` (model inversion), `ML04` (membership inference), and `ML07` (transfer-learning attacks) inform the Data Science and Legal/Governance contracts' scope.
- MITRE ATLAS, <https://atlas.mitre.org/>. Named techniques (`AML.T0051.000` LLM prompt injection, `AML.T0043` craft adversarial input) drive the Safety contract's out-of-cycle review triggers.

Local solutions these contracts land against:

- `mod-112-owning-an-ai-eval-program/exercise-01` (release gate architecture) — the gates whose rules each contract scopes.
- `mod-109-human-review-workflows/exercise-03` (IAA at scale) — Review Ops contract's floor comes from here.
- `mod-110-eval-data-platform-slice/exercise-01` (schema + lineage) — artefact names used in RACI rows.
- `mod-111-cost-latency-quality-tradeoff/exercise-01` (multi-objective weights) — Product contract's weight ownership.
- `mod-108-app-safety-and-guardrails-eval/exercise-*` — Safety contract's suite ownership.
- `mod-107-online-eval-and-regression/exercise-02` (drift alerts) — Data Science contract's out-of-cycle trigger.

<!-- Team names, SLO durations, and signer counts are exercise fixture tied to the running support_agent surface. They illustrate the structure; do not cite as benchmark data. -->
