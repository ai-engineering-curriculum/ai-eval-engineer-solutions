# exercise-03 — Build Vs Buy Eval Platform Matrix — Solution

> Reference solution for `lessons/mod-112-owning-an-ai-eval-program/exercises/exercise-03-build-vs-buy-eval-platform-matrix.md`.
> This is a **design** exercise. The deliverable is a decision matrix (`build-vs-buy-eval-platform-<app-family>.md`) that ranks "build in-house", "adopt an open-source stack", and "buy a commercial eval vendor" across a fixed set of dimensions, with weights, scores, and a recommendation that a VP of Engineering and a CFO can read in one sitting. Numbers in the worked matrix below are **exercise fixture** — illustrative values tied to the running `support_agent` surface, not published vendor data.

## 1. Solution overview

A build-vs-buy matrix is three things:

1. **A set of dimensions** that are mutually exclusive (a dimension is one real question) and jointly exhaustive (nothing load-bearing is missing). Eleven dimensions here, grouped into three families: *fit* (does the option do the job), *risk* (what exposure does it create), and *economics* (what does it cost over 24 months).
2. **A weighting rubric** published in the body of the document, not in a side spreadsheet. Weights sum to 1; a reviewer who disagrees with the recommendation can re-weight and re-score locally.
3. **A tie-break policy.** Two options within 10% of each other on the weighted score are a tie; the tie-break reads a named qualitative factor (usually data residency or governance) rather than another tenth-of-a-point of the same axis.

Five design choices anchor the matrix:

1. **Three options, not two.** "Buy a vendor" without an "adopt OSS" column collapses the real decision (which is usually *"what do we build on top of OSS and what do we outsource"*). The three options force the trade-off to be named, not hand-waved.
2. **Weights are declared by the stakeholder, not the analyst.** The analyst scores; the stakeholder (VP Engineering + CFO + Legal, depending on the dimension family) weights. The matrix publishes both so the reader sees the knob.
3. **Scoring is on a 5-point rubric per dimension, with a named rubric anchor at 1 / 3 / 5.** "5 = best-in-class, 1 = unusable" is not a rubric; it's a sentiment. Anchors make scores auditable.
4. **The cost column is a 24-month TCO with a confidence interval**, not a sticker price. Build cost includes staff; buy cost includes professional services; OSS includes the headcount to run it. The CI reflects uncertainty, not false precision.
5. **The "do nothing" row is a visible option.** "Keep what we have" is sometimes the right answer; it is never the right answer without being written down. Row 0 forces the comparison.

```
                         fit                  risk                 economics
                  ┌────────────┐         ┌────────────┐         ┌────────────┐
                  │ coverage   │         │ data       │         │ 24-mo TCO  │
                  │ depth      │         │ residency  │         │ ramp cost  │
                  │ flexibility│         │ vendor     │         │ switching  │
                  │ time-to-   │         │  lock-in   │         │  cost      │
                  │  value     │         │ security   │         │            │
                  └────────────┘         └────────────┘         └────────────┘
                     weight: 0.35          weight: 0.30           weight: 0.35
                           │                      │                      │
                           └──────────────────────┼──────────────────────┘
                                                  ▼
                                  weighted score per option
                                                  ▼
                                      ▲ tie-break: residency
                                                  ▼
                                          recommendation
```

The matrix is one artefact, 2–3 pages. It is **not** a procurement exercise — vendor-selection RFPs are downstream. The matrix answers "which *path* are we on", not "which specific vendor".

## 2. Implementation — worked answer — `build-vs-buy-eval-platform-support-agent.md`

```markdown
# Build-vs-buy — eval platform for support_agent
Author: <eval lead>  |  Date: 2026-11-19  |  Doc version: 1.0
Stakeholders: VP Engineering (A), CFO (C on economics), Legal (C on risk),
Eval Lead (R), Head of Safety (C on risk).

## 1. The decision

One decision: on what *path* do we build the eval platform for the
`support_agent` family over the next 24 months. Options:

- **A. Build in-house** on top of OpenTelemetry + a managed data warehouse +
  our own gate runner and dashboard. Zero commercial eval vendor.
- **B. Adopt OSS** — Arize Phoenix / Langfuse / Weave as the trace store
  and eval UI, extend with in-house gate runner + judge model harness.
- **C. Buy** — commercial eval vendor as the primary trace store, eval
  runner, dashboards, and governance workflows; integrate with CI.
- **0. Do nothing** — keep the current ad-hoc setup (Python scripts +
  Google Sheets). Baseline.

The matrix below scores all four.

## 2. Dimensions and weights

Weights set by VP Engineering + CFO + Legal in a one-hour session on
2026-11-12 (notes linked). Weights sum to 1.0 across all 11 dimensions.

| # | Family | Dimension | Weight | Who weighted |
|---|---|---|---|---|
| D1 | fit | Coverage — supported artefact types (trace, eval result, dataset, rubric, gate) | 0.08 | VP Eng |
| D2 | fit | Depth — ability to express our rubrics (partial credit, trajectory scoring, human review in loop) | 0.08 | Eval Lead |
| D3 | fit | Flexibility — can we add a new gate or dimension without a vendor PR | 0.07 | Eval Lead |
| D4 | fit | Time to first green gate in CI | 0.07 | VP Eng |
| D5 | fit | Interop with our existing stack (OTel, our CI, our DW) | 0.05 | Eval Lead |
| D6 | risk | Data residency and privacy (PII in traces) | 0.10 | Legal |
| D7 | risk | Vendor lock-in / exit cost | 0.08 | VP Eng |
| D8 | risk | Security posture of the data path | 0.07 | Head of Safety |
| D9 | risk | Governance / audit evidence quality | 0.05 | Legal |
| D10 | economics | 24-month TCO (staff + licenses + infra) | 0.20 | CFO |
| D11 | economics | Switching cost if we pivot later | 0.15 | VP Eng + CFO |
| | | **Total** | **1.00** | |

## 3. Scoring rubric per dimension

Anchor-based 1 / 3 / 5. Anchors (excerpted):

| Dim | 1 (worst) | 3 (adequate) | 5 (best) |
|---|---|---|---|
| D1 Coverage | Traces only; no eval-result or dataset store | Traces + eval results + basic dataset versioning | Full artefact set + lineage + policy registry |
| D2 Depth | Scalar rubric only, no partial credit | Scalar + binary + partial credit; limited trajectory support | Full trajectory + tool-call scoring + human-in-loop |
| D3 Flexibility | Vendor change required for new gate | Config-only but vendor-side approval | We own the gate code; config PR lands in a day |
| D4 Time-to-value | ≥ 6 months to a green gate in CI | 2–3 months to a green gate | 2–4 weeks to a green gate |
| D5 Interop | Proprietary ingest; we write adapters | OTel-compatible ingest; we write dashboards | Full OTel + warehouse export + CI SDK |
| D6 Residency | No residency controls; data leaves region | Region pinning available; PII masking optional | Full residency controls + customer-managed keys |
| D7 Lock-in | Proprietary format; export is lossy | Full export supported but re-ingest is expensive | Open format; swap-out is a weekend |
| D8 Security | No SOC 2; no security review | SOC 2 Type II; standard security review | SOC 2 Type II + continuous pentest + bug bounty |
| D9 Governance | No audit log on config/override changes | Audit log on overrides; no replay | Hash-chained audit log + replayable policy history |
| D10 TCO (24 mo) | > 2× baseline | 1–2× baseline | ≤ 1× baseline (or baseline is the vendor cost) |
| D11 Switching | Full rewrite to leave; ≥ 2 quarters | Partial rewrite; ≥ 1 quarter | Shim swap; days |

24-month TCO is reported with a 90% confidence interval. The CI covers
staff ramp uncertainty (±30% on build headcount), vendor pricing
uncertainty (list → actual negotiated), and growth uncertainty
(trace volume ±2×).

## 4. Scores

Scored on 2026-11-18 by Eval Lead with sign-off from VP Engineering.
Each cell is `<score> (<rationale ≤ 20 words>)`.

| Dim (weight) | A. Build | B. OSS | C. Buy | 0. Do nothing |
|---|---|---|---|---|
| D1 Coverage (0.08) | 5 (we build what we need) | 4 (Phoenix/Langfuse cover most; dataset versioning needs work) | 4 (vendor covers most; gate/policy may be vendor-shaped) | 1 (ad-hoc scripts) |
| D2 Depth (0.08) | 5 | 4 (partial credit possible via custom scorer) | 3 (vendor rubrics are usually scalar-first) | 2 |
| D3 Flexibility (0.07) | 5 | 4 | 2 (new gate requires vendor roadmap) | 5 (it's all our code — but no leverage) |
| D4 Time-to-value (0.07) | 2 (≥ 6 months) | 4 (6–10 weeks) | 4 (6–10 weeks with PS) | 5 (already running, but the running is the problem) |
| D5 Interop (0.05) | 5 | 4 | 3 (OTel ingest varies by vendor) | 2 |
| D6 Residency (0.10) | 5 (our cloud, our rules) | 4 (self-hosted OSS) | 2 (SaaS default; residency add-on may be enterprise-tier only) | 5 |
| D7 Lock-in (0.08) | 5 | 4 | 2 | 5 (no vendor to lock in) |
| D8 Security (0.07) | 3 (we have to pass the review ourselves) | 3 (self-hosted; we own the posture) | 4 (SOC 2 ships) | 1 (Google Sheets is the security posture) |
| D9 Governance (0.05) | 4 (we build it to the module spec) | 3 (OSS tooling exists; audit layer is ours) | 4 (vendor often ships audit logs) | 1 |
| D10 24-mo TCO (0.20) | 3 (2 FTE × 24mo ≈ 1.5× baseline; CI ±30%) | 4 (1 FTE + infra ≈ 1.0× baseline; CI ±25%) | 3 (vendor list + PS + 0.5 FTE ≈ 1.3× baseline; CI ±40%) | 5 (sunk) |
| D11 Switching (0.15) | 4 (leaves clean; we own it) | 4 (OSS; exits by stopping to pay for hosting) | 2 (lossy export; re-ingest ≥ 1 quarter) | 5 (nothing to switch) |

Rationales longer than 20 words go in Appendix A. Confidence intervals
on D10 appear in Appendix B.

## 5. Weighted totals

| Option | Weighted score | 90% CI (from D10/D11 only) | Rank |
|---|---|---|---|
| A. Build | 4.03 | [3.86, 4.17] | 2 |
| B. Adopt OSS | **3.86** | [3.73, 3.98] | 1* |
| C. Buy | 2.96 | [2.74, 3.17] | 3 |
| 0. Do nothing | 3.49 | [3.49, 3.49] | — |

\* Rank 1 by tie-break: Build's weighted score is nominally higher than OSS,
but the CIs overlap (A: [3.86, 4.17] vs B: [3.73, 3.98]). Tie-break (§7)
promotes OSS.

## 6. Recommendation

**Adopt OSS (Option B)** as the backbone, with in-house extensions for
gate rules and policy registry. 24-month path:

- **Months 0–2**: stand up self-hosted Phoenix / Langfuse / Weave as the
  trace + eval-result store. Replace the Google-Sheets scorer with the
  OSS UI. Early green gate for `support_agent` offline eval (G2 from
  ex-01).
- **Months 2–6**: build in-house gate runner + policy registry on top.
  Pin the OSS versions to a known-good release; in-house code is our
  flexibility surface.
- **Months 6–12**: add trajectory + tool-call extensions (mod-103),
  cohort-preserving routing (mod-111), online eval (mod-107). Wire G8/G9
  (canary gates).
- **Months 12–24**: audit log replay (ex-01 §8), model card automation
  (ex-05), incident-regression workflow (ex-04).

Build column (A) is a close second; chosen against on 24-month TCO (D10)
and time-to-value (D4). Buy (C) is ranked third on lock-in (D7),
residency (D6), and flexibility (D3) — three of the four dimensions the
stakeholders weighted highest.

## 7. Tie-break policy

When two options' weighted scores have overlapping 90% CIs, the tie-break
reads the following in order:

1. **Data residency (D6).** The option with the higher D6 wins; this is
   the dimension Legal weighted explicitly for `support_agent`.
2. **Switching cost (D11).** The option that costs less to walk away
   from wins.
3. **Depth (D2).** The option that can express our rubrics without
   degradation wins.

Applied here: D6 tie between A and B (both scored 5 and 4 respectively);
D11 tied at 4. Go to D2 — A scored 5, B scored 4. **A would normally win
the tie-break**.

However, this is where the "do nothing" row and time-to-value (D4) matter:
Option A's 6-month ramp exposes us to a longer period on Option 0's
stack. The stakeholder sign-off (VP Engineering + CFO) explicitly weighted
D4 for that reason. Option B lands a green gate in 6–10 weeks and
preserves optionality to migrate to A later (D11 Build = 4, and OSS →
in-house port is a known-cost migration).

Decision committee: VP Eng (A), Eval Lead (R), CFO (C), Legal (C), Head
of Safety (C). Signed on 2026-11-19.

## 8. Risks and mitigations

- **OSS vendor fades.** Mitigation: pin the version; keep the in-house
  gate runner free of OSS-specific coupling. D11 Build = 4 is the exit.
- **Scale.** OSS projects may not scale to tenant volumes at our
  terminal size. Mitigation: load-test at month 12 against 2× current
  volume; schedule the migration decision then, not after the breach.
- **Governance gap.** OSS audit tooling may not satisfy a future
  regulator. Mitigation: build the hash-chained audit log as part of
  ex-01 §8 ourselves; do not depend on the OSS project for governance
  evidence.
- **Headcount.** The 1-FTE assumption for OSS is load-bearing. If the
  team cannot hire by Q2, revisit against C (Buy) rather than slip the
  plan.

## 9. Governance alignment

- NIST AI RMF `GOVERN 1.1` (legal/regulatory), `GOVERN 1.3` (accountable
  processes), `MAP 3.1` (risk/benefit characterization) — the matrix is
  the characterization document.
- NIST AI RMF `MANAGE 1.3` (risk/benefit documentation) — weights and
  scores are the documentation.
- OWASP ML Top 10 `ML05` (model theft) and `ML06` (AI supply chain) —
  D7 lock-in and D8 security map to these.
- MITRE ATLAS does not drive this matrix directly, but the gates the
  platform runs are informed by ATLAS techniques (see ex-01 §9).

## Appendix A — Score rationales > 20 words

(omitted in this excerpt; one paragraph per cell > 20 words)

## Appendix B — TCO confidence intervals

(omitted in this excerpt; shows build-hours, vendor-list-vs-negotiated,
and growth factor assumptions)

## Appendix C — Weight-sensitivity analysis

If the CFO re-weights D10 from 0.20 to 0.30 (shifting from D3 and D4 by
0.05 each), the ranking becomes: B 3.82, 0 3.59, A 3.94, C 2.84. Option
A still leads nominally; "do nothing" climbs. The recommendation
(Option B) is stable under reasonable weight perturbations within ±0.05
per dimension.
```

## 3. Validation steps

A reviewer validates the submission by:

1. **Count dimensions and weight sum.** 11 dimensions; weights sum to 1.0 exactly. Rounding errors fail — the matrix is not a casual spreadsheet.
2. **Open the rubric table (§3).** Every dimension has anchor text at 1, 3, and 5. "5 = best" with no anchor fails; you cannot audit a judgement that has no reference.
3. **Open the scores (§4).** Every cell has a score and a ≤ 20-word rationale. Blank rationales fail.
4. **Verify the weighted total.** Reproduce one option's total by hand: `sum(weight × score)`. Values match the §5 table to two decimals.
5. **Open the CI column.** D10 and D11 are reported as CIs, not point estimates. The assumption table (Appendix B) names the sources of uncertainty.
6. **Check the "do nothing" row.** Row 0 is visible and scored, not a rhetorical gesture.
7. **Check the tie-break (§7).** When top options have overlapping CIs, the tie-break reads named qualitative factors — not another tenth-of-a-point of the same axis.
8. **Check stakeholder sign-off.** The weighting session date and attendees are named. A matrix with weights chosen by the analyst alone has a conflict-of-interest problem — the recommender should not also set the knobs.
9. **Walk the recommendation (§6).** The 24-month path has named modules and ties to specific exercises in mods 103, 107, 110, 111, 112 so a reader can estimate feasibility.
10. **Sensitivity analysis.** Appendix C shows that a reasonable re-weighting (±0.05 per dim) does not flip the recommendation. If the ranking flips under tiny perturbations, the matrix is not decisive and the recommendation is premature.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Three real options + "do nothing" row. Four rows visible. | A, B, C, 0 all scored. | Only two options (build vs buy). |
| R2 | 10–12 dimensions, grouped into fit / risk / economics. | Group headers visible; dimensions MECE. | Flat list; overlapping dimensions. |
| R3 | Weights sum to exactly 1.0; stakeholder sign-off named. | Sum = 1.00; session date + attendees. | Weights off by rounding; analyst-only weighting. |
| R4 | Scoring rubric has anchors at 1 / 3 / 5 per dimension. | Anchor text present per dim. | 1–5 with no anchors. |
| R5 | Every cell has a score + ≤ 20-word rationale. | Rationale present; longer in Appendix A. | Blank cells; no rationale. |
| R6 | TCO is reported as a 24-month CI, not a sticker. | 24 mo horizon; CI on D10/D11. | Point-estimate sticker price. |
| R7 | Tie-break policy named; applied when CIs overlap. | Section 7 present and applied. | Nominal-winner wins; no tie-break. |
| R8 | Recommendation includes a 24-month path with named modules / exercises. | §6 has months 0–24 milestones. | "We recommend X" with no plan. |
| R9 | Weight-sensitivity analysis shows stability under reasonable perturbations. | Appendix C present. | No sensitivity; recommendation is a point estimate. |
| R10 | Governance alignment names NIST / OWASP / ATLAS entries that map to specific dimensions. | §9 maps each standard to a dim. | "Aligned with NIST" without mapping. |

A passing submission is 8+ of 10 with **R3, R4, R5, R6, R7** hard-required.

## 5. Common mistakes

- **Two-option matrix.** "Build vs buy" without an OSS column collapses the real decision. Three options surface the trade-off.
- **No "do nothing" row.** Baseline is always a candidate; refusing to score it means the status quo wins by default without being evaluated.
- **Analyst weights the dimensions.** The analyst's weights will match the analyst's conclusion. Weighting is a stakeholder activity; scoring is an analyst activity.
- **5-point scale without anchors.** "5 = best" turns into "5 = whichever option I liked when I wrote the row". Anchor text makes scores reproducible.
- **Point-estimate TCO.** A sticker price hides the ±30% staff-ramp uncertainty and the list-vs-negotiated vendor pricing. Report a CI; the recommendation has to survive the CI, not just the point.
- **Dimensions overlap.** "Flexibility", "customisation", and "extensibility" are one dimension with three names; score them as three and you triple-count. MECE.
- **Economics weight dominates.** If `D10 + D11 > 0.5` the matrix is an economics memo in disguise. Rebalance or acknowledge it in the title.
- **Tie-break reads score magnitudes, not qualitative factors.** "A wins by 0.11 so pick A" ignores that the CI spans 0.14. The tie-break is a named qualitative column (residency, exit cost, depth).
- **Recommendation is a sentence.** "We recommend OSS" with no ramp plan is unactionable. The 24-month path is the real deliverable; the matrix is the justification.
- **Risks section missing.** Every option has risks. A matrix that lists only the winning option's upsides fails the reader; §8 must name the real failure modes.
- **No sensitivity analysis.** If a ±0.05 reweighting flips the winner, the recommendation is a coin flip with extra steps.
- **Confusing vendors with paths.** The matrix answers "build / OSS / buy"; it does not pick a specific vendor. Vendor selection is a downstream RFP, with its own rubric.

## 6. References

Official standards:

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework>. The matrix is organised against `GOVERN 1.1` (legal/regulatory), `GOVERN 1.3` (accountable processes), `MAP 3.1` (risk/benefit characterization), `MANAGE 1.3` (trade-off documentation). The weighting session and the sensitivity analysis are the MANAGE 1.3 artefacts.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/>. `ML05` (model theft) and `ML06` (AI supply chain) inform dimensions D7 (lock-in) and D8 (security posture). A vendor that cannot answer D8's anchor questions is scoring itself on OWASP ML Top 10 guidance.
- MITRE ATLAS, <https://atlas.mitre.org/>. Not a direct input to this matrix, but the gates the platform runs are shaped by ATLAS techniques; the matrix therefore scores the platform's ability to *carry* ATLAS-informed gates (D2 Depth and D3 Flexibility).

Local solutions this matrix reads from:

- `mod-102-trace-instrumentation/exercise-02` (Phoenix/Langfuse/Weave bake-off) — the OSS column (B) inherits findings from here.
- `mod-110-eval-data-platform-slice/exercise-01` (schema + lineage), `exercise-03` (multi-runner orchestration), `exercise-05` (CI + online integration) — artefact coverage for D1 reads these.
- `mod-111-cost-latency-quality-tradeoff/exercise-01` (multi-objective report) — scoring extension D2 depth depends on.
- `mod-112-owning-an-ai-eval-program/exercise-01` (release gate architecture) — the gate tree the chosen path has to host.

<!-- Scores, weight values, and 24-month ramp milestones in §2–§6 are exercise fixture tied to the running support_agent surface. They illustrate the structure of the matrix; do not cite as vendor benchmarks. -->
