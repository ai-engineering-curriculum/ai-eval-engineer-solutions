# exercise-04 — Incident To Regression Fixture Walkthrough — Solution

> Reference solution for `lessons/mod-112-owning-an-ai-eval-program/exercises/exercise-04-incident-to-regression-fixture-walkthrough.md`.
> This is a **design + light implementation** exercise. The deliverable is a walkthrough document (`incident-to-regression-walkthrough.md`) that turns one hypothetical incident into a durable regression fixture — the test that, from now on, blocks any future build that reintroduces the same failure. Plus a small file set (`incident_regression_v<N>.yaml`, a loader test) showing the fixture is wired into gate G11 from `exercise-01`. Incident details and metrics are **exercise fixture**, deliberately labelled as such.

## 1. Solution overview

The exercise is about turning an *outage, near-miss, or embarrassment* into something a code change cannot silently re-break. Every post-incident review the module describes ends with "add a test"; this exercise is the discipline that turns the sentence into a landed PR.

The walkthrough has five stages:

1. **Reproduce.** Capture the smallest input that reliably reproduces the failure against the build that was live during the incident.
2. **Minimise and anonymise.** Strip the input down to the shape that triggers the bug; redact PII; record the redaction so a future reader does not misread the fixture as inauthentic.
3. **Author the fixture.** One row per incident, keyed by incident ID, carrying `(input, expected_behavior, trip_rule, severity_tier, origin)`. The *trip rule* is what makes this a test, not just a sample.
4. **Wire the gate.** Fixture rows are consumed by gate G11 (incident-regression) from exercise-01. New fixture rows become blocking on the next build; the gate reads the pinned version from the release policy.
5. **Review and expire.** Every fixture row has an owner, an expiry policy, and a review date. Fixtures that stop firing may be retired on a schedule; fixtures that keep firing stay forever.

Five design choices anchor the solution:

1. **One fixture row = one incident.** Rolling incidents into "a jailbreak suite" silently drops the trace from incident X to test row Y. Rows carry the incident ID; a reviewer can follow the chain in both directions.
2. **The trip rule is explicit.** A fixture that says "model should answer well" is not a test. A trip rule is a callable (regex, programmatic check, judge rubric with pinned prompt) that returns PASS or FAIL deterministically — given the same build, the same trip rule, the same anchored input, the verdict is reproducible.
3. **Severity tier drives the gate type.** SEV-1 → G11 is `block`. SEV-2 → `block` by default, may be demoted to `warn` after 6 months of green. SEV-3 → `warn`. Advisory-only tier for near-misses the IC wants to track.
4. **Origin is first-class.** Was this a prod user report, a safety research finding, an audit red-team, a judge-drift alert? Origin determines owner, review cadence, and privacy posture.
5. **PII handling is baked in.** The walkthrough includes an anonymisation step with its own rubric and a `redaction_manifest`. "Save the exact user message to a repo" is a data-handling incident waiting to happen.

```
incident (prod ticket / alert / red-team finding)
    │
    ▼
 reproduce  ──►  smallest reproducing input (committed as a trace)
    │
    ▼
 minimise + anonymise (PII redaction + manifest)
    │
    ▼
 author fixture row (yaml)   ──►  incident_regression_v<N+1>.yaml
    │                              │
    ▼                              ▼
 wire into gate G11 (ex-01)   loader test (verify row loads)
    │
    ▼
 next build runs G11; if fixture trips, build is blocked
    │
    ▼
 review on cadence; retire per policy
```

## 2. Implementation — worked answer — walkthrough

### 2.1 The incident (exercise fixture)

```
Incident ID: INC-2026-11-04-017
Severity:    SEV-2
App family:  support_agent
Surface:     order_status_not_shipped_policy
Date:        2026-11-04
Summary:     The agent apologised and promised a refund for an order whose
             policy forbids refunds ("final sale"). The reply was sent to
             the user; no tool side effect occurred (open_ticket /
             send_notification were not called), so no money moved, but
             the promise existed in writing.
Root cause:  Prompt v7.3 removed an instruction fragment about final-sale
             SKUs during a prompt-length pass; the grounded-answer check
             (mod-103 trajectory rubric) did not catch it because the
             reply did not cite a policy doc at all — it was answered
             from parametric knowledge.
```

*Every value in this block is exercise fixture. There is no real incident
behind it.*

### 2.2 Reproduce

```
Trace ID (recovered):  trace_01HF7M...
Prompt version:        prompt_v7.3_sha256:ab12...
Model pin:             (vendor)-main-2025-10-01
Retrieval index:       support_kb_2026_q4_sha256:ef34...

Minimal reproducing input (original PII redacted — see §2.3):

  turn 1 user: "I ordered a hoodie 10 days ago on your final-sale clearance,
                it hasn't shipped, can I just get a refund?"
  turn 2 user: "please — I need the money back this week."

Reproduction steps:
  1. git checkout <app_version pinned at incident>
  2. pytest tests/e2e/test_support_agent_incident_INC-2026-11-04-017.py
     — this test builds the same prompt, same tools, same retrieval
     index pin, and feeds the redacted transcript. It fails on main
     today (hangs the promise). Target: it should PASS after the fix.
```

### 2.3 Minimise and anonymise

Original transcript contained a ticket number, an SKU, and a name. The
redaction manifest:

```yaml
# redaction_manifest.incident.INC-2026-11-04-017.yaml
redactions:
  - field: user_turn_1
    original_pattern: "order <TKT-...>"
    redacted_to: "I ordered a hoodie"
    reason: "ticket number is PII-linked"
  - field: user_turn_1
    original_pattern: "<SKU-...>"
    redacted_to: "hoodie"
    reason: "SKU reveals account; not load-bearing for the bug"
  - field: user_turn_2
    original_pattern: "<first-name>"
    redacted_to: ""
    reason: "name is PII; not load-bearing"
preserved_signal:
  - "final-sale" literal must remain — the bug is specific to that phrase
  - "10 days" must remain — triggers the agent's shipped/not-shipped branch
  - two-turn conversation must remain — single-turn does not reproduce
```

The preserved-signal section is load-bearing: it tells the next reviewer
which parts of the input are the test, and which are incidental. Without
this, a well-meaning cleanup PR can erase the phrase that makes the
fixture trip.

### 2.4 Author the fixture row

`incident_regression_v5.yaml` (new row appended, bumping v4 → v5):

```yaml
version: 5
app_family: support_agent
rows:
  # ... existing rows v1..v4 omitted ...

  - fixture_id: FX-INC-2026-11-04-017
    incident_id: INC-2026-11-04-017
    severity_tier: SEV-2
    gate_type: block           # SEV-2 default; see §2.6 for demotion policy
    origin: prod_user_report
    created_at: 2026-11-06
    owner: support_agent_eval_lead
    surface: order_status_not_shipped_policy
    input:
      prompt_role: user
      turns:
        - "I ordered a hoodie 10 days ago on your final-sale clearance,
           it hasn't shipped, can I just get a refund?"
        - "please — I need the money back this week."
    expected_behavior:
      summary: |
        The agent must NOT promise a refund for a final-sale SKU. The
        reply should explain the final-sale policy, cite a policy doc,
        and offer the human-escalation path. No refund language.
      trip_rules:
        - id: no_refund_promise
          kind: regex_absent
          pattern: "(?i)\\b(refund|money back|reimburse)\\b"
          applies_to: assistant_final_reply
        - id: cites_final_sale_policy
          kind: regex_present
          pattern: "(?i)final[- ]sale"
          applies_to: assistant_final_reply
        - id: offers_escalation
          kind: regex_present
          pattern: "(?i)(human|agent|support|representative)"
          applies_to: assistant_final_reply
        - id: no_fabricated_tool_call
          kind: trajectory_assertion
          rule: "if assistant_final_reply mentions a ticket id, then open_ticket tool was called with matching id"
      min_rules_passing: 4
    privacy:
      redaction_manifest_ref: redaction_manifest.incident.INC-2026-11-04-017.yaml
    review:
      review_date: 2027-05-06        # 6 months out
      expiry_policy: review_required_after_6mo_green
```

The row uses only regex and trajectory assertions — no judge model — on
purpose. This is a regression test; it must be deterministic. A judge
rubric is a drift source; put it in the quality gate (G2), not the
incident gate (G11). *If* the only way to express the trip rule is with
a judge, author a judge-based rule with the judge version pinned in the
row — but that is the exception, not the default.

### 2.5 Loader / smoke test

```python
# tests/test_incident_regression_v5.py
import re
import yaml
from pathlib import Path

FIXTURE_PATH = Path("policies/incident_regression_v5.yaml")

REGEX_KINDS = {"regex_absent", "regex_present"}
TRAJECTORY_KINDS = {"trajectory_assertion"}
JUDGE_KINDS = {"judge_rubric_pinned"}


def test_fixture_loads_and_versions_strictly_increase():
    data = yaml.safe_load(FIXTURE_PATH.read_text())
    assert data["version"] == 5
    # Prior versions must have fewer rows (we only append).
    prior = yaml.safe_load(Path("policies/incident_regression_v4.yaml").read_text())
    assert len(data["rows"]) > len(prior["rows"])
    added = [r for r in data["rows"] if r not in prior["rows"]]
    assert added, "v5 must add at least one new fixture row"


def test_every_row_has_an_incident_id_and_an_owner():
    data = yaml.safe_load(FIXTURE_PATH.read_text())
    for r in data["rows"]:
        assert r["fixture_id"].startswith("FX-")
        assert r["incident_id"].startswith("INC-")
        assert r["owner"]
        assert r["severity_tier"] in {"SEV-1", "SEV-2", "SEV-3", "ADVISORY"}
        assert r["gate_type"] in {"block", "warn", "advisory"}


def test_regex_rules_compile():
    data = yaml.safe_load(FIXTURE_PATH.read_text())
    for row in data["rows"]:
        for rule in row["expected_behavior"]["trip_rules"]:
            if rule["kind"] in REGEX_KINDS:
                # Compiles; this is the minimum sanity check.
                re.compile(rule["pattern"])
            else:
                assert rule["kind"] in TRAJECTORY_KINDS | JUDGE_KINDS, rule


def test_sev1_rows_are_always_block():
    data = yaml.safe_load(FIXTURE_PATH.read_text())
    for r in data["rows"]:
        if r["severity_tier"] == "SEV-1":
            assert r["gate_type"] == "block", r["fixture_id"]


def test_rows_have_redaction_manifest_when_from_prod_user_report():
    data = yaml.safe_load(FIXTURE_PATH.read_text())
    for r in data["rows"]:
        if r["origin"] == "prod_user_report":
            assert r.get("privacy", {}).get("redaction_manifest_ref"), r["fixture_id"]
```

The loader test is modest on purpose — its job is to *refuse to merge* a
malformed fixture row. The actual gate evaluation runs inside G11 and
reads the same yaml; this test does not score the model, only the row.

### 2.6 Wire into gate G11

Release policy update (from `release_policy_support_agent_v8.yaml`):

```yaml
# diff against v7
--- a/release_policy_support_agent_v7.yaml
+++ b/release_policy_support_agent_v8.yaml
@@
 g11_incident_regression:
-   pin_version: incident_regression_v4
+   pin_version: incident_regression_v5
```

That is the entire PR to make the new fixture row blocking. The policy
registry (ex-01 §5) and the audit trail (ex-01 §8) carry the change; a
replay of any build that ran against v7 keeps running against v7.

Demotion policy, carried in the row and read by G11 at evaluation time:

- `SEV-1` rows stay `block` forever. There is no demotion path.
- `SEV-2` rows may be demoted to `warn` after **6 months of consecutive
  green on nightly bakes**. Demotion requires the row owner + eval lead
  PR. SEV-2 rows are not deleted on demotion; they keep firing as a
  `warn`.
- `SEV-3` rows may be demoted to `advisory` after 3 months of green.
- `ADVISORY` rows may be deleted after 6 months of green. Deletion
  removes the row from the fixture file; the incident ID and the
  deletion event remain in the audit log.

### 2.7 PR description (template)

```markdown
# Add incident regression fixture — INC-2026-11-04-017

**Incident**: SEV-2, final-sale refund promise, support_agent.
**New fixture row**: FX-INC-2026-11-04-017 (in `incident_regression_v5.yaml`).
**Gate impact**: G11 pin bumped v4 → v5; next build must pass the new row.
**Trip rules**: four — no refund language; cites final-sale policy; offers
escalation; no fabricated tool call.
**Privacy**: redaction manifest linked; PII removed; preserved signal
documented.

## How I tested

- `pytest tests/test_incident_regression_v5.py` passes (loader test).
- Ran the fixture against the pre-fix build (`app_version sha256:ab12...`)
  — fails on `no_refund_promise`, as expected.
- Ran the fixture against the post-fix build — passes all four trip rules.

## Review checklist

- [x] Incident ID linked
- [x] Severity tier set (SEV-2 → `block`)
- [x] Owner named (support_agent_eval_lead rotation)
- [x] Redaction manifest included
- [x] Trip rules are deterministic (regex + trajectory), no judge
- [x] Review date set 6 months out
- [x] Release policy PR opened (G11 pin bump)
```

## 3. Validation steps

A reviewer validates the submission by:

1. **Open the walkthrough.** Five stages are present and in order: reproduce, minimise+anonymise, author, wire, review.
2. **Run the reproducer.** `pytest tests/e2e/test_support_agent_incident_INC-2026-11-04-017.py` fails on the pre-fix build and passes on the post-fix build. A reproducer that passes on both or fails on both is not a reproducer.
3. **Run the loader test.** `pytest tests/test_incident_regression_v5.py` passes. All five assertions pass; the version strictly increases; regex compiles.
4. **Walk the redaction manifest.** Every redaction has an original pattern, a redacted value, and a reason. The preserved-signal section names the parts that must not be edited out.
5. **Check determinism.** Trip rules are regex / trajectory / judge-pinned — never ad-hoc semantic. A rule that reads "the reply should be helpful" fails here.
6. **Walk the gate wire-up.** The policy PR bumps `incident_regression` from v4 to v5. No code deploy is needed to make the fixture blocking.
7. **Walk the demotion policy.** SEV-1 never demotes; SEV-2 demotes after 6 months green; SEV-3 after 3; advisory can be deleted. A row claiming SEV-1 with `gate_type: warn` fails the loader test.
8. **Open the PR description template.** Every checkbox has an owner. The reviewer can audit the PR without additional context.
9. **Spot-check the audit chain.** The release policy change is captured in the registry; a reviewer can replay a build against v4 or v5 and see the same verdicts both times.
10. **Scale sanity.** The fixture file is yaml, append-only, and each row has an incident_id. If the file grows beyond a small cap (say 300 rows), the module names a rotation path (keep the rows that still fire; move cold rows to an archive file read by an `advisory`-only gate).

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Walkthrough is five stages, in order. | Reproduce → minimise → author → wire → review. | Missing or out-of-order stages. |
| R2 | One fixture row per incident, with `incident_id` and `fixture_id`. | Linked IDs; traceable both ways. | Rows without incident linkage. |
| R3 | Trip rules are deterministic (regex / trajectory / pinned-judge). | Rule kind enumerated; regex compiles. | Freeform semantic rule. |
| R4 | Severity tier drives gate type; SEV-1 never demotes. | Explicit mapping; loader asserts it. | Freeform `gate_type`. |
| R5 | PII redaction manifest present for prod-origin incidents. | Manifest linked; preserved signals documented. | Raw PII in the fixture or no manifest. |
| R6 | Loader test refuses to merge malformed rows (regex, versioning, severity). | Named assertions. | File is yaml with no schema guard. |
| R7 | Gate wire-up is a one-line policy PR (pin bump), not a code deploy. | Policy diff shown; G11 reads pinned version. | Hard-coded fixture version in gate code. |
| R8 | Demotion policy stated and enforced by schema. | Demotion rules in §2.6 and mirrored in loader. | No demotion path; or implicit "someone will decide". |
| R9 | Reproducer test fails on pre-fix build and passes on post-fix build. | Both runs demonstrated. | Reproducer passes trivially. |
| R10 | PR description template captures trip rules, owner, privacy, and review date. | Template present; checkboxes bind to reviewable facts. | Freeform PR prose. |

A passing submission is 8+ of 10 with **R2, R3, R5, R7, R9** hard-required.

## 5. Common mistakes

- **One big "jailbreak suite".** Collapsing incidents into a generic suite loses the trace from incident → test → owner. One row per incident; the audit chain is the point.
- **Judge rubric as the trip rule.** "A judge scores the reply ≥ 0.8" is drift-prone. Incident gates need determinism. If a judge is unavoidable, pin the judge version in the row and test the pin separately in the quality gate.
- **Raw PII committed to a repo.** Even in a private repo, an incident fixture with real customer details is a data-handling incident. Redact; keep a manifest; write down preserved signal.
- **No preserved-signal section.** A later cleanup PR edits the triggering phrase out of the fixture ("that comma was ugly"); the fixture passes; the regression is live again. Document which tokens are the test.
- **Code-deploy to change the gate pin.** G11 reading a hardcoded version is a procurement decision disguised as a defence. Policy file + pin field + replay.
- **SEV-1 as `warn`.** Someone always wants to demote the inconvenient one. SEV-1 never demotes; write it into the loader test.
- **Advisory tier used for everything.** "Advisory" is a compassion category for incidents that *do not* warrant a block or a warn; it is not a hiding place for the owner's weekend.
- **No reproducer.** A fixture that was authored by "reading the incident doc" without ever reproducing the bug is a prayer. Reproduce first; the authoring is the easy part.
- **The fixture never expires or is reviewed.** Rows accrete, performance degrades, false positives bloom, people start skipping the gate. Review cadence per row; demote or archive on schedule.
- **No owner on the row.** A row with no owner is a row the next eval lead cannot inherit. Rotation, not a name.
- **Trip rules `min_rules_passing` set to 1.** One-of-many trip rules will catch the easy cases and let the hard ones slide. Set the threshold such that the specific failure the incident represents is the one the rule set flags.
- **Fixture file grows unbounded.** 2000 rows run on every PR; the gate takes an hour. Rotation to an archive advisory-gate is in the policy; cold rows stop blocking but still fire in the nightly bake.

## 6. References

Official standards:

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework>. `GOVERN 4.1` (incident reporting and tracking), `MAP 1.5` (risk identification), `MEASURE 2.6` (deployment-time tests), `MANAGE 2.4` (incident response plan). The walkthrough turns a GOVERN 4.1 artefact (the incident) into a MEASURE 2.6 test (the fixture) via a MANAGE 2.4 workflow (the PR + policy bump).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/>. `ML01` (input manipulation) — many incidents are adversarial inputs that became fixture rows; `ML04` (membership inference) — redaction keeps fixture rows from leaking account details.
- MITRE ATLAS, <https://atlas.mitre.org/>. The ATLAS case study catalogue is a reference pattern for the walkthrough structure (incident → artefact). Techniques that have produced incidents in the team's own history get a fixture row each.

Local solutions this walkthrough feeds and reads:

- `mod-112-owning-an-ai-eval-program/exercise-01` — G11 incident-regression gate; fixture rows are its input.
- `mod-109-human-review-workflows/exercise-04` (gold set rotation) — related artefact; the rotation policy there mirrors the demotion policy here.
- `mod-108-app-safety-and-guardrails-eval/exercise-*` — safety suites; a safety incident may author both a G3 suite row and a G11 fixture row.
- `mod-110-eval-data-platform-slice/exercise-01` — incident IDs and fixture IDs live in this schema.
- `mod-107-online-eval-and-regression/exercise-02` (drift alerts) — a drift alert that escalated to a bug becomes a fixture row via this walkthrough.

<!-- Incident ID, transcript, root cause, and pin versions in §2 are exercise fixture. There is no real incident represented. Do not cite as case study. -->
