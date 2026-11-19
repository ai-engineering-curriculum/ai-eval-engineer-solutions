# exercise-01 — Release Gate Architecture For One App Family — Solution

> Reference solution for `lessons/mod-112-owning-an-ai-eval-program/exercises/exercise-01-release-gate-architecture-for-one-app-family.md`.
> This is a **design** exercise. The deliverable is a single architecture doc (`release-gate-architecture-<app-family>.md`) that names every gate protecting one app family, says who owns each gate, states the decision rule in a form a reviewer can audit, and lands override policy in writing. The worked answer is anchored on the running `support_agent` surface used across modules 101–111. Treat any named failure mode, metric threshold, or workflow step as **exercise fixture**, not a vendor claim.

## 1. Solution overview

A release gate architecture is three things, not one:

1. **A gate tree** — an ordered set of checks, each with a stable ID, that every candidate build walks through before it can reach production. Gates are organised by *where they fire* (CI, pre-prod canary, prod rollout) rather than by *who authored them*, because the release engineer reads the tree in time order, not org-chart order.
2. **An ownership map** — one named team per gate, one on-call rotation per gate, one SLO per gate (how long an answer may take). Gates without an owner silently become the eval lead's problem, which is the failure mode the module is written against.
3. **A decision ledger** — the gate outcomes, plus the input hashes they ran on, are written to a durable store so a post-incident reviewer can replay the decision. A gate that cannot be audited is a vibe, not a gate.

Five load-bearing choices anchor the design:

1. **Gates are typed `block | warn | advisory`.** `block` fails the pipeline; `warn` fails only when combined with another `warn` on the same build (two yellows → red); `advisory` surfaces a signal without gating. Promoting a warn to a block is a one-line config change; it must not require a rebuild.
2. **Every gate has a ground-truth input and a verdict type.** Offline gates read `(dataset_version_id, evaluator_version_id)` and emit `PASS|FAIL|ERROR`. Online gates read `(window_start, window_end, cohort_filter)` and emit the same. The verdict schema is shared with mod-110 exercise-01 so the dashboard and the audit log read the same rows.
3. **Hard caps are expressed on CI bounds, not point estimates.** A latency gate reads "p95 95% CI lower bound ≤ SLO" rather than "p95 point estimate ≤ SLO". One noisy canary should not fail an otherwise good build, and one lucky canary should not pass a bad one. This matches NIST AI RMF `MEASURE 2.3` (metric reliability) and `MEASURE 2.7` (measurement validity).
4. **Override has a pager, a timer, and a receipt.** Any override pages the eval lead's rotation (not a Slack DM), expires after a named duration (7 or 14 days), and leaves an audit row that the next gate run reads. "Expired override still in effect" is itself a gate.
5. **One app family, one gate tree.** The architecture is scoped to a single product surface (`support_agent` here). A platform team may host the runner, but the gate tree, thresholds, and overrides are owned by the app team. This is deliberate — a one-size gate tree across many app families is the failure mode mod-112 chapter 03 names as "centralised eval tax".

```
candidate build
      │
      ▼
┌─────────────────────────────┐   CI phase — runs on every PR / main merge
│ G1  schema / lineage gate   │   block on missing dataset_version_id
│ G2  offline quality gate    │   block on paired Δ < floor (mod-106)
│ G3  safety / guardrail gate │   block on jailbreak suite fail (mod-108)
│ G4  cost ceiling            │   block on $/1k over cap (mod-111)
└─────────────────────────────┘
      │  all PASS
      ▼
┌─────────────────────────────┐   pre-prod phase — runs on a staging bake
│ G5  shadow eval             │   block on shadow Δ regression (mod-107)
│ G6  trajectory / tool gate  │   block on tool-floor floor (mod-103)
│ G7  human review gate       │   block if IAA < floor on sampled set (mod-109)
└─────────────────────────────┘
      │  all PASS
      ▼
┌─────────────────────────────┐   prod rollout phase — runs during canary
│ G8  canary online quality   │   block on confidence sequence crossing (mod-107)
│ G9  canary latency SLO      │   block on p95 CI over SLO (mod-111)
│ G10 cohort preservation     │   warn if any slice regresses (mod-111)
│ G11 incident-regression set │   block on any new failure from mod-112 ex-04 set
└─────────────────────────────┘
      │  all PASS
      ▼
   general availability
```

The tree is a DAG, not a sequence — several gates run in parallel within a phase — but the phase boundaries are strict: no pre-prod gate runs until every CI gate passes, and no prod gate runs until every pre-prod gate passes. This is how a cost or latency regression caught in CI saves the human reviewer's time in pre-prod.

## 2. Implementation — worked answer — `release-gate-architecture-support-agent.md`

```markdown
# Release gate architecture — support_agent app family
Author: <eval lead>  |  Date: 2026-11-19  |  App family: support_agent (order-status / returns)
Doc version: 1.0  |  Owners-of-record: see §4

## 1. Scope and non-goals

- **In scope**: every gate that protects a `support_agent` release — prompt bump,
  model pin change, tool-catalog change, retrieval index change, judge version
  bump, dataset refresh.
- **Out of scope**: platform-level release gates (image scanning, SBOM, SOC 2
  controls) — those are owned by the Platform Security team and run upstream of
  G1. This document names them in §6 for completeness but does not re-own them.
- **One app family**: support_agent only. The sister doc for `copilot` reuses
  the same gate IDs but may pick different thresholds and owners.

## 2. Gate tree

### 2.1 CI phase (runs on every PR and every main-branch merge)

| ID  | Gate | Input | Verdict rule | Type | Owner | SLO |
|-----|------|-------|--------------|------|-------|-----|
| G1  | Schema + lineage | `(app_version_id, prompt_version_id, dataset_version_id, judge_evaluator_version_id)` | every ID resolves in `mod-110-ex-01` tables; dataset_version is not deprecated | `block` | Eval Platform | 2 min |
| G2  | Offline paired quality | `(dataset_version_id, evaluator_version_id)` vs last GA build | paired Δ quality 95% CI lower bound ≥ `-0.01` (no stat-sig regression) | `block` | Eval Lead | 20 min |
| G3  | Safety / guardrail suite | `jailbreak_suite@v<pin>`, `pii_leak_suite@v<pin>` | pass rate on each suite ≥ floor (per suite) | `block` | Safety | 20 min |
| G4  | Cost ceiling | trace cost sum over offline eval set | `cost_per_1k_requests` ≤ `cost_cap_usd_per_1k` from `release_policy` | `block` | Eval Lead (policy set with Finance) | 5 min |

### 2.2 Pre-prod phase (runs on staging bake, nightly on main)

| ID  | Gate | Input | Verdict rule | Type | Owner | SLO |
|-----|------|-------|--------------|------|-------|-----|
| G5  | Shadow eval | last 24 h prod traffic shadowed through candidate | paired quality Δ 95% CI lower bound ≥ `-0.01`; cohort preservation (see G10) | `block` | Eval Lead | 6 h |
| G6  | Trajectory / tool gate | fixture trajectories from `mod-103-ex-01` surface | tool-floor floor (`wrong_tool`, `unauthorised_tool`) pass rate = 100% | `block` | Eval Lead | 30 min |
| G7  | Human review gate | 50 sampled traces from shadow (stratified by cohort) | IAA (Cohen's κ) ≥ floor from `mod-109-ex-03`; disagreement triage closed | `block` | Review Ops | 24 h |

### 2.3 Prod rollout phase (runs during 1% → 10% → 50% → 100% canary)

| ID  | Gate | Input | Verdict rule | Type | Owner | SLO |
|-----|------|-------|--------------|------|-------|-----|
| G8  | Canary online quality | live sample of traces, scored online | confidence sequence (`mod-107-ex-04`) does not cross lower bound | `block` | Eval Lead | continuous |
| G9  | Canary latency SLO | trace.duration_ns on canary slice | p95 95% CI lower bound ≤ `p95_slo_ms` (from `mod-110-ex-04`) | `block` | SRE | continuous |
| G10 | Cohort preservation | per-cohort Δ on paid-tier, enterprise, locale-fr, locale-ja | no cohort Δ 95% CI entirely below `-0.02` | `warn` (two warns in a build → block) | Eval Lead | continuous |
| G11 | Incident regression set | `incident_regression_v<pin>` from `mod-112-ex-04` | pass rate = 100% (every regression test green) | `block` | Eval Lead | 30 min |

### 2.4 Gate dependencies

```
G1 → {G2, G3, G4}         # schema is prerequisite for everything
{G2, G3, G4} → {G5, G6}   # CI gates precede pre-prod
{G5, G6} → G7             # human review runs on shadow output
{G7} → {G8, G9, G10, G11} # prod rollout only after pre-prod GA
```

Parallelism inside a phase is allowed; phase boundaries are strict.

## 3. Verdict schema

Shared with `mod-110-ex-01`. Every gate writes one row:

```json
{
  "gate_run_id":      "grn_01HF...",
  "build_id":         "app_version_id sha256:...",
  "gate_id":          "G5",
  "phase":            "pre_prod",
  "started_at_ns":    1731987210000000000,
  "finished_at_ns":   1731991000000000000,
  "verdict":          "PASS|FAIL|ERROR",
  "rule":             "paired_delta_ci_lower >= -0.01",
  "observed":         { "paired_delta": -0.004, "ci": [-0.012, 0.004], "n": 842 },
  "override_id":      null,
  "dataset_version":  "sha256:...",
  "evaluator_version":"sha256:...",
  "cohort_filter":    null,
  "run_actor":        "ci@github",
  "notes":            ""
}
```

A `FAIL` with `override_id != null` is counted as a **failed gate with an
override in effect**; it does not pass the pipeline unless the override row
is live (unexpired, not revoked). The override is itself audited.

## 4. Ownership and on-call

| Owner | Gates | On-call rotation | Business-hours SLO | Off-hours SLO |
|---|---|---|---|---|
| Eval Platform | G1 | `eval-platform-oncall` | 2 min (automated) | n/a (automated) |
| Eval Lead | G2, G4, G5, G6, G8, G10, G11 | `support-agent-eval-oncall` | 1 h ack, 1 business day resolve | 4 h ack (block), next business day resolve |
| Safety | G3 | `safety-oncall` | 1 h ack, 1 business day resolve | 4 h ack (block), next business day resolve |
| Review Ops | G7 | `review-ops-oncall` | 1 business day | next business day |
| SRE | G9 | `sre-oncall` | 15 min ack (block), 1 h resolve | 15 min ack (block), 1 h resolve |

Named on-call rotations map to pager groups in the incident tool. A gate
without a rotation is a gate with no owner; such a gate is not allowed to
exist (see §9 Common mistakes).

## 5. Threshold registry

Thresholds are **not** committed into gate code. They live in a single
`release_policy_support_agent_v<N>.yaml` that the eval lead edits via PR.
Format (excerpt):

```yaml
version: 7
effective_at: 2026-11-19T00:00:00Z
app_family: support_agent
policy:
  g2_offline_paired_quality:
    delta_ci_lower_floor: -0.01
  g3_safety:
    jailbreak_suite: { min_pass_rate: 1.00 }
    pii_leak_suite:  { min_pass_rate: 1.00 }
  g4_cost:
    cost_cap_usd_per_1k: 2.50
  g7_human_review:
    min_iaa_kappa: 0.70
    min_sampled_n: 50
  g9_latency:
    p95_slo_ms: 2500
    p99_slo_ms: 6000
  g10_cohort_preservation:
    cohorts: [paid_tier, enterprise, locale_fr, locale_ja]
    per_cohort_delta_ci_lower_floor: -0.02
  g11_incident_regression:
    pin_version: incident_regression_v5
```

Threshold changes require a PR + two reviewers (eval lead + the gate owner).
The PR template requires a one-line rationale per changed threshold. The
previous policy version is archived, not deleted; a reviewer can replay a
past build against a past policy.

## 6. Upstream gates (reference only)

These are owned by other teams and fire *before* G1. They are listed so the
app team's reviewer knows which gates have already fired:

- Image scan + SBOM (Platform Security)
- Dependency vulnerability scan (Platform Security)
- Secret scan (Platform Security)
- Infra IaC policy (Platform SRE)

If any upstream gate fails, the pipeline never reaches G1. The eval lead
does not need to re-check these.

## 7. Overrides and exception policy

### 7.1 When an override is allowed

- A gate failure that the eval lead has triaged and has a signed statement
  of *why the signal is noise or why the risk is acceptable for a bounded
  window*.
- A rollback: if G11 catches a regression that was already in production,
  an override on G2–G7 may be allowed to ship a *fix* without re-running
  the full pre-prod bake. The eval lead signs; the override expires in 72h.

### 7.2 When an override is NOT allowed

- `G3` safety / guardrail override requires **both** eval lead and safety
  lead signatures. No single-signer override on safety.
- `G11` incident-regression override requires the eval lead plus the IC
  of the original incident. No single-signer override on an incident test.
- `G9` latency override requires SRE lead signature.
- An override cannot be used to ship a build that fails `G2` quality
  against a dataset that has been refreshed in the same PR — "I refreshed
  the dataset and the gate fired" is dataset manipulation, not an override
  case.

### 7.3 Override record

```json
{
  "override_id":      "ovr_01HF...",
  "gate_id":          "G7",
  "build_id":         "app_version_id sha256:...",
  "signers":          ["evl-123", "revops-77"],
  "reason":           "...one paragraph, no shorthand...",
  "expires_at_ns":    1733123456789000000,
  "linked_incident":  "INC-2026-11-15-042",
  "revocable_by":     ["eval-lead", "safety-lead"]
}
```

Every override generates a calendar reminder to the signer two days before
expiry. On expiry, the override row flips to `expired`; the next gate run
that reads it treats the build as `FAIL`.

## 8. Audit trail and replay

- Every `gate_run` is immutable (write-once, hash-chained to the previous
  run for the same build).
- The eval platform surfaces `v_release_history_support_agent` (one row per
  GA build, with the IDs of every gate run that let it through).
- Replay: `aicg gate replay --build <id> --policy <version>` re-runs the
  gate tree against a (possibly older) policy version and emits a diff.
  This is the artefact a post-incident review reads.

## 9. Governance alignment

- **NIST AI RMF** `MANAGE 1.3` — document the trade-offs each threshold
  encodes. Threshold registry §5 carries one line of rationale per entry.
- **NIST AI RMF** `MEASURE 2.3`, `MEASURE 2.7` — gates evaluate CIs, not
  point estimates, and name the measurement method (bootstrap vs paired
  bootstrap vs confidence sequence) explicitly in the `rule` field.
- **NIST AI RMF** `GOVERN 1.2`, `GOVERN 4.1` — ownership map §4 names a
  team and a rotation per gate.
- **OWASP ML Top 10** `ML01` input manipulation / `ML02` data poisoning —
  G1 lineage gate enforces that the dataset_version_id is not a locally-
  mutated copy; a build that points at a dataset version the registry does
  not have is blocked.
- **OWASP ML Top 10** `ML05` model theft / `ML06` supply chain — upstream
  SBOM/image-scan gates (§6) cover model and dependency supply.
- **MITRE ATLAS** `AML.T0051.000` LLM prompt injection — G3 safety gate
  includes a published jailbreak suite; the test set version is pinned.
- **MITRE ATLAS** `AML.T0043` craft adversarial input — G11 incident
  regression set carries the adversarial inputs that have escaped past
  builds (see mod-112 ex-04 walkthrough).

## 10. Open questions (for the next doc revision)

- `G10` cohort preservation currently uses a per-cohort CI floor; the
  mod-111 ex-02 note on cohort-preserving routing recommends a paired-
  cohort test with a higher power — follow-up ticket filed.
- `G11` incident regression set is currently binary (pass/fail); next
  revision should tier `block` vs `warn` by incident severity (SEV-1
  incidents stay `block`, SEV-3 may drop to `warn`).
```

## 3. Validation steps

A reviewer validates the submission by:

1. **Open `release-gate-architecture-support-agent.md` and read §2.** Every gate has an ID, an input, a verdict rule, a type, an owner, and an SLO. A gate missing any of the six fields is incomplete.
2. **Trace a candidate build through the tree.** Pick a hypothetical change (e.g. "prompt v8 → v9 on `support_agent`"), and walk the tree: which gates run, in what order, who pages on failure. If the walk exits the tree with "someone figures it out", the tree is incomplete.
3. **Resolve every threshold.** Open `release_policy_support_agent_v<N>.yaml` and check that every gate rule in §2 that reads a threshold reads it from §5, not a hardcoded value. A threshold that is not in the registry is a hidden gate.
4. **Walk the override policy.** Each override case has a signer list, an expiry, and a prohibited case. "The eval lead can always override" is not an override policy.
5. **Check the audit trail.** `gate_run` schema is present, immutable, and hash-chained. A reviewer can answer "which gates let build X through, against which policy version, on which dataset" without reading code.
6. **Check phase boundaries.** No pre-prod gate runs until all CI gates pass; no prod gate runs until all pre-prod gates pass. A pre-prod gate that can fire before CI completes is a design bug.
7. **Check governance alignment.** Each cited standard (NIST AI RMF function, OWASP ML Top 10 entry, MITRE ATLAS technique) maps to a specific gate ID. "Covers NIST" without naming a function or a gate is not an alignment.
8. **Check scope discipline.** The doc covers one app family. Thresholds for `support_agent` do not leak into a shared `platform.yaml`. A doc that implicitly covers every app at the company is the centralised-tax failure mode.
9. **Dry-run a failure drill.** Pick a gate (e.g. G5 shadow eval) and walk the on-call response: who pages, what SLO applies, what the resolution path is. If the on-call rotation is "the eval lead by name", the doc is one bus accident away from being obsolete.
10. **Reproducibility.** `aicg gate replay --build <id> --policy <version>` re-runs the tree on an archived build + policy. Byte-identical output is the bar.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Gate tree is phased (CI → pre-prod → prod rollout) with strict phase boundaries. | Phases named; phase boundary rules explicit. | Flat list of gates with no phase. |
| R2 | Every gate has `{ID, input, verdict rule, type, owner, SLO}`. | All six fields per gate. | Gate missing an owner, rule, or type. |
| R3 | Verdict schema is shared with upstream trace / eval schema (mod-110 ex-01). | Same field names; reviewer can `JOIN`. | Each gate writes its own ad-hoc log. |
| R4 | Thresholds live in a versioned policy file, not in gate code. | Registry §5 present; PR-reviewed. | Hardcoded thresholds; policy changes require code deploy. |
| R5 | Rules evaluate CIs (or confidence sequences), not point estimates, for noisy axes. | `*_ci_lower >= floor` style rules. | `latency_p95 <= 2500` style rules. |
| R6 | Override policy has signers, expiry, prohibited cases, and an audit record. | Four items present; multi-signer on safety and incident gates. | "The eval lead signs" is the whole policy. |
| R7 | Ownership map has a team **and** a rotation per gate (not a named person). | Pager group named per gate. | Named individuals only. |
| R8 | Audit trail: gate runs are immutable, hash-chained, and replayable against an archived policy. | Replay CLI described; byte-identical output. | No replay path. |
| R9 | Each gate maps to at least one named NIST AI RMF function / OWASP ML entry / MITRE ATLAS technique in §9. | Mapping §9 present and specific. | Standards named without mapping. |
| R10 | Document is scoped to one app family; upstream platform gates referenced but not re-owned. | §1 scope + §6 upstream reference. | Doc implicitly covers every app. |

A passing submission is 8+ of 10 with **R2, R4, R5, R6, R7** hard-required.

## 5. Common mistakes

- **One big score.** Rolling G2 through G11 into a single "release readiness score" collapses the audit — a failing safety gate and a failing cost gate cannot net to the same number. Keep gates separate; let the pipeline do the AND.
- **Point-estimate thresholds on noisy axes.** "p95 ≤ 2500ms" fails a canary that happened to hit 2501ms on a 60-sample window. Evaluate the CI; fail only when the CI floor is still over the cap.
- **Hardcoded thresholds in gate code.** A finance-driven cost cap change becomes a code deploy. Keep thresholds in a versioned policy file; the policy PR is the governance surface.
- **"Eval lead signs" as the whole override policy.** A single-signer override on a safety or incident gate is how a near-miss ships to production. Multi-signer on safety and incident gates is not negotiable.
- **No phase boundaries.** Running G7 (human review) before G2 (offline quality) wastes reviewer time on a build that the offline eval would have failed in 20 minutes.
- **No gate for the regression set from past incidents.** Without G11, an incident fix reverts silently on the next refactor. The incident set has to be a *blocking* gate, not an advisory one.
- **Owner = a person's name.** The eval lead changes job titles; the gate is still there. The owner is a rotation, not a person.
- **No override expiry.** "Permanent override on G7" is the story every post-mortem tells. Overrides expire; expiry is a gate.
- **Centralised gate tree across every app family.** A single `release_policy.yaml` for every app forces one team to approve every threshold change for every product. The result is that nobody owns anything; module chapter 03 names this the centralised-tax failure mode.
- **Replay is a nice-to-have.** A gate that cannot be replayed on an archived policy version cannot support a post-incident review. The replay CLI is part of the design, not a follow-up.
- **Missing the data lineage gate (G1).** Without it, a candidate that trained on a leaked dataset version or a human-authored "eval" that the eval lead has never seen can walk the tree. G1 is the cheapest gate and the one most often forgotten.
- **Safety tests as `warn` to "unblock the team".** `warn` on safety is how you ship a jailbreak. Safety gates are `block`, with a published alternative path (fix the tests or fix the model).

## 6. References

Official standards:

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `GOVERN 1.2` (accountability roles), `GOVERN 4.1` (incident reporting), `MEASURE 2.3` (metric reliability), `MEASURE 2.7` (measurement validity), `MANAGE 1.3` (risk/benefit documentation). The gate tree's ownership map (§4), the CI-bound thresholds (§2), and the threshold registry (§5) are organised around these functions.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML01` input manipulation, `ML02` data poisoning, `ML05` model theft, `ML06` supply chain. G1 (lineage), G3 (safety suite), and the upstream SBOM / image-scan gates (§6) map to these entries.
- MITRE ATLAS, <https://atlas.mitre.org/> — technique catalogue for adversarial ML; `AML.T0051.000` (LLM prompt injection) and `AML.T0043` (craft adversarial input) are the first two techniques G3 and G11 defend against. The incident-regression set in G11 is intended to grow with each ATLAS technique the app family sees in production.

Local solutions this gate tree depends on:

- `mod-106-eval-gated-cicd/exercise-*` — the CI harness G2–G4 plug into.
- `mod-107-online-eval-and-regression/exercise-01` (sampled online eval), `exercise-03` (canary + shadow launch gates), `exercise-04` (confidence sequences) — G5, G8 implementations.
- `mod-108-app-safety-and-guardrails-eval/exercise-*` — G3 test suites.
- `mod-109-human-review-workflows/exercise-03` (IAA at product scale) — G7 floor.
- `mod-110-eval-data-platform-slice/exercise-01` (schema + lineage), `exercise-04` (SLOs), `exercise-05` (CI + online integration) — the verdict schema, SLO table, and gate-run store this doc reads from.
- `mod-111-cost-latency-quality-tradeoff/exercise-01` (multi-objective report), `exercise-04` (latency) — G4, G9, G10 implementations.
- `mod-112-owning-an-ai-eval-program/exercise-04` (incident → regression fixture) — G11 input.

<!-- Threshold values, cohort names, and workflow steps in §2–§5 are exercise fixture tied to the running support_agent surface. They are not vendor benchmarks and must not be cited as such. -->
