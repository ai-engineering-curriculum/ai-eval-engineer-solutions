# exercise-04 — Public Benchmark Shape-Template Audit — Solution

> Reference solution for `lessons/mod-103-trajectory-and-tool-eval/exercises/exercise-04-public-benchmark-shape-template-audit.md`.
> Chapter 06 is the primary input; mod-101 exercise-04's leaderboard audit is the upstream pattern. The audit is anchored on the running `support_agent` surface — a tool-using customer-support assistant with the chapter-02 argument matcher, chapter-03 rubric, chapter-04 budgets, and chapter-05 Inspect harness defined in exercises 01–03 of this module.

## 1. Solution overview

The deliverable is a single markdown file (`benchmark-audit-support-agent.md`, reproduced in §2 below) that audits three public agent benchmarks against the `support_agent` surface. One `accept`, one `adapt`, one `reject` — chosen on purpose so the stakeholder memo has to defend a non-trivial decision, not collect three rejects.

Four design choices anchor the solution:

1. **Audit shape, not quality.** The question is *"does this benchmark's shape plug into this module's scorer stack?"* — not *"is this a good benchmark?"*. A useful shape can come from a benchmark whose leaderboard number you would not cite.
2. **Pick one accept, one adapt, one reject.** `τ-bench` is the obvious accept (same shape — airline / retail customer-support agents, database verifier, `pass^k`); BFCL is the plausible adapt (AST-based function-call evaluator is reusable as a Level-4 upgrade to chapter 02); GAIA is the honest reject (final-answer exact-match on multi-step research questions is a different surface from a database-mutating support agent).
3. **Every accept / adapt names the chapter slot.** Chapter 02 Level-N argument matcher, chapter 03 final-answer scorer, chapter 04 reporting metric, chapter 05 Solver. If a shape adoption cannot name the slot, it is vapour.
4. **Every reject gets a specific alternative.** "Use τ-bench's retail split instead of GAIA" is a citeable recommendation; "GAIA is not a fit" is not.

## 2. Implementation — worked answer — `benchmark-audit-support-agent.md`

```markdown
# Benchmark audit — support_agent surface
Author: <you>  |  Date: 2026-11-06  |  Surface version: support_agent@v4.1

## 1. Surface recap

- **Surface**: support_agent, a tool-using customer-support assistant for an
  e-commerce order-status / returns workflow.
- **Task families (production)**: `order_status_not_shipped_policy`,
  `returns_in_window_refund`, `escalate_to_human`, `faq_no_tools`.
- **Tool catalog**: `docs.search` (policy retrieval), `crm.lookup`
  (order / customer record), `open_ticket` (CRM mutation), `send_notification`
  (email dispatch — guarded by mod-108 safety eval).
- **Final-answer shape**: short natural-language reply that often *references*
  a mutated side effect ("I've opened ticket TCK-12345 with reason=…"). It is
  neither a pure text answer (GAIA shape) nor a pure state mutation (WebArena
  shape).
- **Rubric weight profile** (v1.4): name 0.20, args 0.10, order 0.10, answer
  0.20, groundedness 0.25, budget 0.15. Floors: wrong_tool, unauthorised_tool,
  ordering_violation, fabricated_claim. Pass threshold 0.80.

## 2. Per-benchmark audit

### 2.1 τ-bench (Yao et al., arXiv:2406.12045)

| # | Question | Answer |
|---|---|---|
| 1 | **Task schema** | τ-bench's airline and retail tasks are customer-support agents interacting with a database via function calls and a simulated user. The `(input, reference)` shape is `(simulated_user_persona + initial_message, expected_final_db_state + policy-compliance assertions)`. This *matches* the support_agent surface shape — database-mutating conversations with a user turn count and a terminal state the scorer can read. |
| 2 | **Scoring function** | τ-bench scores on terminal DB state plus a hand-written list of policy assertions. Reusable in chapter 02's argument matcher slot at Level 2 (exact-value on terminal state) and Level 3 (semantic-equivalence on the assertion list). The DB-state scorer is an almost drop-in substitute for the open_ticket reference verifier. |
| 3 | **Metric family** | τ-bench reports `pass^k` — the probability that `k` independent runs all pass. This is a reliability metric; it fits the support_agent rollout gate as a *second* axis next to chapter-04's budget-family rollups. For a canary a PM reads, `pass^1 = pass_rate` is the headline; `pass^8` is the reliability tail you watch when the agent is deployed in production. |
| 4 | **Harness compatibility** | τ-bench ships a reference implementation (Python, uses an `agents-interact` runtime). Porting to an Inspect `Task` is a half-day job: the `Sample` input becomes the initial user message, the tool catalog becomes Inspect `@tool` stubs, and the terminal DB-state assertion becomes a scorer. The user simulator belongs in a `Solver` that handles turn-taking. |
| 5 | **Contamination** | Public dataset; released mid-2024. Any frontier model trained after that date has likely seen the examples. The *shape* is reusable regardless; the leaderboard number is suspect for new frontier models. For the support_agent surface the dataset is replaced with the surface's own tasks anyway. |
| 6 | **Judge dependence** | None — τ-bench's scoring is programmatic (DB-state comparison + rule-based policy assertions). The user simulator uses an LLM, but it is upstream of scoring, not inside the scorer. |
| 7 | **What to leave behind** | The dataset (airline/retail distribution ≠ our e-commerce distribution), the specific policy rules, and the leaderboard number. |
| — | **Verdict** | **accept** |

### 2.2 Berkeley Function-Calling Leaderboard (BFCL)

| # | Question | Answer |
|---|---|---|
| 1 | **Task schema** | BFCL's `(input, reference)` is `(natural-language request, expected function-call signature)`. Shape is narrower than support_agent (single-turn tool call rather than a trajectory) but it is exactly the per-step name+arguments sub-problem chapter 02 names in isolation. |
| 2 | **Scoring function** | BFCL uses an AST-based evaluator for function-call arguments (<https://gorilla.cs.berkeley.edu/leaderboard.html>). This is a direct Level-4 upgrade for chapter 02's argument matcher — the chapter stubs Level 4 as an escape hatch for tools whose arguments are code, and BFCL's evaluator is the open-source reference implementation. |
| 3 | **Metric family** | Per-category accuracy (`simple`, `multiple`, `parallel`, `multi_turn`). The accuracy family is not a sensible shape for rollout gates (chapter 04 uses p95 + flagged-verdict rates, not accuracy), so BFCL's reporting does not plug in as-is. The underlying AST evaluator does. |
| 4 | **Harness compatibility** | BFCL ships Python evaluation code; the AST comparator is importable. No Inspect Task required — the AST comparator is used *inside* exercise-01's argument scorer, not run as a separate Task. |
| 5 | **Contamination** | Benchmark is well-known; frontier models have been fine-tuned on function-calling and plausibly on BFCL itself. The leaderboard number is contaminated; the AST evaluator is not. |
| 6 | **Judge dependence** | None. |
| 7 | **What to leave behind** | The dataset, the per-category reporting, the leaderboard number. Adopt only the AST-comparison code. |
| — | **Verdict** | **adapt** |

### 2.3 GAIA (Mialon et al., arXiv:2311.12983)

| # | Question | Answer |
|---|---|---|
| 1 | **Task schema** | GAIA tasks are multi-tool research questions with an exact-match short final answer. The `(input, reference)` shape is `(research question, short-string answer)` — single-turn, non-interactive, no state mutation. This does *not* match the support_agent surface shape: our tasks mutate state (open tickets, send notifications) and the reply references the mutation. |
| 2 | **Scoring function** | Exact-match on the normalised final answer. Chapter 03's `exact_match_normalised` already implements this shape; nothing to borrow. |
| 3 | **Metric family** | Pass@1 per difficulty level. Appropriate for GAIA; wrong shape for a rollout gate on a surface where the trajectory matters. |
| 4 | **Harness compatibility** | A community Inspect port exists; the scoring is a one-liner. The adoption cost is near zero, which is seductive, but it is also beside the point — the shape does not match the surface. |
| 5 | **Contamination** | Known; GAIA is public. Not the deciding factor here. |
| 6 | **Judge dependence** | None (exact-match). |
| 7 | **What to leave behind** | Everything. GAIA is the right benchmark for a research-agent surface; it is the wrong shape for a customer-support surface. |
| — | **Verdict** | **reject** |

## 3. Shape-template adoption plans

### 3.1 τ-bench — accept

- **Borrowed**: the terminal-DB-state scoring function (feeds chapter 02 Level-2/3 argument matcher and chapter 03 final-answer scorer); the `pass^k` reliability metric (feeds chapter 04 reporting as a second axis).
- **Replaced**: the dataset (τ-bench's airline/retail → support_agent's e-commerce order-status tasks); the policy rule list; the leaderboard number.
- **Chapter slot**:
  - **Chapter 02**: τ-bench's DB-state verifier plugs in as a specialised argument matcher for `open_ticket` / `send_notification` — the tools whose arguments determine terminal state. The existing Level-3 token-set matcher on `docs.search.query` is unchanged.
  - **Chapter 03**: τ-bench's policy-assertion shape plugs into the `answer` scorer slot as an alternative to `exact_match_normalised`, raised when the task family's final-answer-is-a-reference-to-a-mutation shape is in play.
  - **Chapter 04**: `pass^k` joins `pass_rate` on the rollout gate dashboard. mod-107 owns the implementation depth.
  - **Chapter 05**: The simulated user becomes an Inspect `Solver` that handles turn-taking; the tool catalog uses Inspect `@tool` stubs with the same side-effect discipline as exercise-03.
- **Module owner**: mod-103 owns the scorer extension; mod-107 owns the `pass^k` reporting; mod-110 owns the versioned fixture that captures the new assertion shape.
- **Fixture versioning**: a new `reference_v4_0.yaml` under `support_agent/order_status_not_shipped_policy` carrying the DB-state assertion block. Rubric `v1.5` adds a `terminal_state_assertions` floor to `trip_on_verdict` so a mutated-wrong-state failure is a hard-fail regardless of the reply's text.
- **Smoke test**: re-score five historical trajectories from the past week — one per branch (shipped, not_shipped_policy, not_shipped_escalate, not_found, faq_no_tools) — with the new scorer; confirm the pass rate on the mutated-state branches moves from "pass because the reply text was fine" to "fail because the terminal state did not match".
- **Verdict taxonomy entries fed**: `fabricated_claim` (the reply references a ticket that was never opened), `extra_tool` (the agent called `send_notification` when the branch said to escalate silently), `ordering_violation` (CRM lookup after ticket open).

### 3.2 BFCL — adapt

- **Borrowed**: the AST-based function-call evaluator — specifically the argument-comparison code — as the chapter 02 Level-4 implementation for tools whose arguments contain code-shaped literals (we have one on the roadmap: a `sql_query_review` tool for power users).
- **Replaced**: the dataset, the per-category reporting, the leaderboard number. The BFCL benchmark itself does not run against the support_agent surface.
- **Chapter slot**:
  - **Chapter 02**: Level-4 argument matcher. Currently a `NotImplementedError` stub in exercise-01; the BFCL AST comparator fills the stub.
  - **Chapter 03**: unchanged — the AST matcher's output is a per-argument match boolean, which rolls up through the existing `args_credit` axis.
- **Module owner**: mod-103 owns the scorer; mod-110 owns the per-tool rule registry that selects Level 4.
- **Fixture versioning**: adds an `argument_rules.sql_query_review.query: {level: 4-ast}` entry to the next `reference_vX.Y` when the tool goes live.
- **Smoke test**: author 10 `sql_query_review` call pairs that differ only in whitespace and alias names; verify the AST matcher returns `match=True` where the Level-2 exact matcher returns `match=False`. Author 5 pairs that are semantically different (different `WHERE` clause); verify the AST matcher returns `match=False`.
- **Verdict taxonomy entries fed**: `argument_mismatch` (false-fails that AST eliminates).

## 4. Stakeholder memo

```
To: <PM>
From: <you>
Subject: Public-benchmark audit for support_agent — three verdicts

You asked whether we should adopt τ-bench, BFCL, or GAIA against support_agent.

Three bullets, one per benchmark:

- **τ-bench — accept.** Shape matches our surface exactly: database-mutating
  customer-support conversations with programmatic scoring. We would borrow
  the DB-state verifier and the `pass^k` reliability metric; we would NOT
  cite τ-bench's leaderboard number as a baseline, because the frontier
  models in it have been trained on it. Chapter-slot: Level-2/3 argument
  matcher + chapter-04 `pass^k` reporting.

- **BFCL — adapt.** Shape is narrower than ours — single-turn function calls
  rather than trajectories — but the AST-based argument evaluator is the
  open-source reference implementation of chapter 02's Level-4 matcher, and
  we need it for the upcoming `sql_query_review` tool. Chapter-slot: Level-4
  argument matcher.

- **GAIA — reject.** Exact-match on a short final answer is the right shape
  for research-agent benchmarks; it is the wrong shape for a surface where
  the trajectory mutates state. Our chapter-03 `exact_match_normalised` is
  already the one-line scorer GAIA wants; there is nothing left to borrow.
  **Specific alternative**: use τ-bench's retail split as the shape template
  for an interim public comparison; the dataset is not our distribution but
  the shape matches.

Next step: I will have the τ-bench shape-template adoption plan for the
`order_status_not_shipped_policy` task family ready by Thursday. Happy to
walk the plan through at the next eval review.
```

## 3. Validation steps

A reviewer validates the submission by:

1. **Reading the three primary sources** — τ-bench's arXiv abstract + methods, the BFCL leaderboard page + sister paper on AST evaluation, GAIA's arXiv abstract + sample tasks. Any audit row whose answer contradicts the primary source fails.
2. **Checking the seven questions are full-sentence**, not bullet lists. A one-line "yes" to "task schema match" is not an audit; chapter 06 §Starter guidance says "two paragraphs about a benchmark you have not read is not an audit; it is an opinion".
3. **Confirming every `accept` and `adapt` names the chapter slot**. A plan that says "we would reuse τ-bench" without saying *where* in chapters 02–05 it plugs in is incomplete.
4. **Confirming every `reject` names a specific alternative**. GAIA → τ-bench retail; AgentBench → τ-bench airline; WebArena → an internal state-mutation task (surface-dependent); etc.
5. **Checking the memo is under 300 words**, structured as one-sentence recap + three bullets + alternative + next step.
6. **Smoke-running the adopted τ-bench shape on five historical trajectories** (stretch — but the smoke test is the bar that separates an audit from a wish list).

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Three benchmarks audited; one accept, one adapt, one reject. | Mix present; not three of the same verdict. | Three rejects, or three accepts. |
| R2 | Each benchmark's seven questions answered in full sentences, with the primary source cited. | Full sentences; arXiv / project URL on first mention. | Yes/no answers; no citation. |
| R3 | Every `accept` and `adapt` names the chapter slot (02 Level-N / 03 final-answer / 04 reporting / 05 Solver). | Slot named explicitly. | "We would use it" without placement. |
| R4 | Every `reject` names a specific alternative (usually another audited benchmark). | Alternative named. | Reject with no fallback. |
| R5 | The adoption plans distinguish borrowed vs replaced; dataset / leaderboard-number / judge are explicit about what is NOT adopted. | All three called out. | "Adopt τ-bench" with no scoping. |
| R6 | Stakeholder memo ≤ 300 words, structured per the exercise spec. | Word count in range; structure matches. | Over length; or academic exchange tone. |
| R7 | Verdict taxonomy (`fabricated_claim`, `extra_tool`, …) referenced where adopted shapes feed specific flags. | At least one chapter-02/03 verdict named per adopted shape. | Taxonomy not referenced. |
| R8 | Audit is explicit about contamination for each benchmark, and distinguishes dataset contamination from shape reusability. | Contamination answer per benchmark is specific. | Contamination handwaved. |
| R9 | Audit was written against the primary sources, not the project's marketing pages. | arXiv / project repo URLs cited. | Secondary summaries only. |
| R10 | File naming + structure: `benchmark-audit-<surface-slug>.md`; four sections in order. | Matches. | Does not. |

A passing submission is 8+ of 10 with R2, R3, R4, R6 hard-required.

## 5. Common mistakes

- **Rejecting on distribution alone.** "τ-bench is airline/retail, we are e-commerce support — reject" throws away the shape with the dataset. Shape is independent of distribution; the dataset is always replaced. Chapter 06 §Starter guidance flags this.
- **Accepting on leaderboard appeal.** "The best model gets 80% on BFCL so if we use BFCL we can quote that number" — no, you cannot, because the leaderboard number is contaminated and your tool catalog is not BFCL's. Adopt the shape; cite your surface's number.
- **Writing three rejects.** Chapter 06 §Starter guidance is explicit: pick on purpose so at least one is arguable. Three rejects is the easy case and skips the auditing skill the exercise builds.
- **Confusing "adapt" with "accept with tweaks".** An `adapt` is "borrow one component, discard the rest"; an `accept` is "plug in largely as-is, dataset-replacement aside". If you are borrowing more than you discard, it is an accept; if you are discarding more than you borrow, it is an adapt.
- **Forgetting the "where it plugs in" bullet.** Every adoption plan names a chapter-N slot: a Level-N argument matcher, a chapter-03 final-answer scorer, a chapter-04 reporting metric, a chapter-05 Solver. An adoption plan that cannot name the slot is a slogan.
- **Writing the memo like an academic abstract.** The audience is a PM who reads to the end; use short sentences and a defended verdict per bullet. Save the methods section for the audit above.
- **Reading a benchmark's project page instead of the primary paper.** Marketing pages overstate the fit; the methods section tells you whether the scorer is programmatic or judge-based, and whether the metric is pass@1 / pass^k / win-rate. The audit depends on knowing which.
- **Treating "the benchmark ships an Inspect Task" as a decisive factor.** It is a nice-to-have, not a decision driver. Harness compatibility saves time; it does not make a bad-shape fit.

## 6. References

Primary sources (all cited above):

- τ-bench — Yao, S., et al., "τ-bench: A Benchmark for Tool-Agent-User Interaction in Real-World Domains", arXiv:2406.12045, <https://arxiv.org/abs/2406.12045>.
- Berkeley Function-Calling Leaderboard — Gorilla / UC Berkeley, <https://gorilla.cs.berkeley.edu/leaderboard.html>.
- GAIA — Mialon, G., et al., "GAIA: A Benchmark for General AI Assistants", arXiv:2311.12983, <https://arxiv.org/abs/2311.12983>.
- SWE-bench Verified (mentioned in the surface recap for completeness) — <https://openai.com/index/introducing-swe-bench-verified/>, Jimenez et al., arXiv:2310.06770, <https://arxiv.org/abs/2310.06770>.
- WebArena — Zhou et al., arXiv:2307.13854, <https://arxiv.org/abs/2307.13854>.
- AgentBench — Liu et al., arXiv:2308.03688, <https://arxiv.org/abs/2308.03688>.
- ToolBench / ToolLLM — Qin et al., arXiv:2307.16789, <https://arxiv.org/abs/2307.16789>.

Official standards:

- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> (motivates distinguishing contamination from shape reusability — a contaminated benchmark cannot ground capability claims but its shape is still legitimate).
- MITRE ATLAS, <https://atlas.mitre.org/> (ATLAS technique catalogue informs which verdict taxonomy entries an adopted shape should feed).
- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> (Measure / Manage functions inform the versioning discipline on adopted shapes — "if you cannot name the version you adopted, you cannot audit the gate").

Local chapters:

- `lessons/mod-103-trajectory-and-tool-eval/06-public-benchmarks-as-shape-templates.md` (primary input for this audit).
- `lessons/mod-103-trajectory-and-tool-eval/02-scoring-per-step-tool-calls.md` (chapter-02 slots that τ-bench and BFCL plug into).
- `lessons/mod-103-trajectory-and-tool-eval/03-final-answer-and-partial-credit-rubric.md` (chapter-03 slot for τ-bench's assertions).
- `lessons/mod-103-trajectory-and-tool-eval/04-budgets-and-rollout-gates.md` (chapter-04 `pass^k` reporting).
```

## 3. Validation steps

A reviewer validates the submission as follows:

1. **Open `benchmark-audit-support-agent.md` and read §1 Surface recap.** Five lines; names task families, tools, final-answer shape, rubric weight profile. If any of the five anchors is missing, the rest of the audit is unanchored.
2. **Read each benchmark's audit table.** Every row is a full sentence (not "yes" or "n/a"); every answer cites the primary source on first mention.
3. **Spot-check contamination answers.** The contamination row distinguishes "dataset is public and seen by frontier models" (which does not disqualify shape reuse) from "the leaderboard number is contaminated" (which does disqualify citing the number as capability).
4. **Open each adoption plan.** Confirm the "where it plugs in" bullet names a specific chapter-N slot and a specific module (`mod-103`, `mod-105`, `mod-108`, `mod-110`, `mod-111`). A plan that cannot name the slot and the module owner is incomplete.
5. **Read the stakeholder memo.** Word-count under 300; one-sentence recap; three bullets; a specific alternative on the reject; a dated next step.
6. **Cross-reference verdict-taxonomy mentions** (`fabricated_claim`, `extra_tool`, `ordering_violation`, etc.) against the chapter-03 taxonomy. Each adopted shape explicitly names which taxonomy entries it would strengthen or weaken.
7. **Optional smoke run** — if the stretch goal was done, five historical trajectories were re-scored with the adopted shape; the pass-rate shift is reported.

## 4. Rubric / review checklist

Reproduced from §2's `## 4. Rubric / review checklist` section inside the deliverable markdown. The ten-row checklist with R2, R3, R4, R6 hard-required applies to this solution as written. No duplicate rubric is maintained here — the deliverable is the markdown file; the rubric is part of it.

## 5. Common mistakes

Reproduced from §2's `## 5. Common mistakes` section inside the deliverable. The nine mistakes listed cover the realistic failure modes for this exercise. Two emphasised:

- **Rejecting on distribution alone.** Chapter 06 is explicit — shape ≠ dataset. "Not our distribution" rejects every benchmark on principle and skips the audit.
- **Writing the memo like an academic abstract.** The audience is a PM. Short sentences, defended verdicts, a next step.

## 6. References

See §2's `## 6. References` section inside the deliverable. The same primary sources and official standards are cited here with no duplication.
