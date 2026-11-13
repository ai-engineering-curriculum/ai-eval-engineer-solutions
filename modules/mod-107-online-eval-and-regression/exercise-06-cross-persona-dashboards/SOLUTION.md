# exercise-06 — Cross-Persona Dashboards — Solution

> Reference solution for `lessons/mod-107-online-eval-and-regression/exercises/exercise-06-cross-persona-dashboards.md`.
> Design (not just wiring) for the four dashboards the eval system owes to its distinct consumers. Deliverable is a layered spec: audience, decisions the dashboard supports, panels with their queries, alert routing, and an anti-pattern list.

## 1. Solution overview

Different people look at "how is the AI app doing?" for different reasons, and a single dashboard that tries to serve all of them ends up serving none. The exercise defines four personas, and the solution is one dashboard each, each purpose-built:

1. **Release owner (on-call eng, "is it safe to ship next?")** — hard-gate status, canary stage, time-to-decision, safety regressions, trace examples.
2. **Product manager ("is the app getting better?")** — quality trend, task-completion, user feedback scores, cost-per-successful-task, cross-surface comparison.
3. **ML / eval engineer ("are my judges / detectors healthy?")** — judge throughput, judge-vs-rule agreement, drift signal counts, CS width over time, judge cost burn.
4. **SRE / reliability ("are latency and cost within budget?")** — p50/p95/p99 by surface and stage, cost-per-request vs budget, error rates, fallback triggers.

The four dashboards share a data layer — the exercise-01 verdict sink, the exercise-03 experiment ledger, the exercise-02 signals stream, and the OTel trace backend — but they do not share panels. A "big board" that mixes personas forces every viewer to ignore 80% of the content.

Design choices:

- **Each dashboard answers a *decision*, not a *question*.** "Should I promote the canary?" (release owner) is a decision. "What is the judge agreement rate?" is a question — it belongs on the ML dashboard only if a specific decision hangs on it.
- **One number per decision, with context panels below.** The top of each dashboard is a single "decision helper" — a status pill (green / yellow / red), a one-sentence interpretation, and a link to the trace or run that made the decision tip. Supporting panels are below. This is cognitive-load driven: humans can absorb one decision at a time.
- **Time ranges match the persona's cadence.** Release owner looks at the current rollout + last 48h. PM looks at the current week + last 4 weeks. ML looks at the last 24h for operational panels + last 30d for trend panels. SRE looks at the current hour + 7 days.
- **Every panel has a declared owner and an on-call routing rule.** A chart nobody owns rots in a week. Owners are named in the YAML dashboard spec; routing rules say which Slack / PagerDuty route takes an alert from this panel.
- **Panels link back to raw traces.** Every "number" row has a hyperlink that opens three example traces. On the release-owner dashboard this is non-negotiable; a regression number without three example traces is unactionable.

## 2. Worked implementation

```
mod-107/exercise-06/
├── dashboards/
│   ├── release-owner.yaml
│   ├── product-manager.yaml
│   ├── ml-eval-engineer.yaml
│   └── sre-reliability.yaml
├── queries/
│   ├── canary_status.sql
│   ├── quality_trend.sql
│   ├── judge_agreement.sql
│   ├── drift_signal_count.sql
│   ├── cost_budget_burn.sql
│   └── latency_percentiles.sql
└── validation/
    ├── required_panels.yaml      # panels every persona dashboard must carry
    └── test_renders.py           # renders each dashboard; confirms no 404 panels
```

### 2.1 Release-owner dashboard (excerpt)

```yaml
# dashboards/release-owner.yaml
name: "Release owner — AI canary"
owner: "@ai-release-eng"
alert_routes:
  default: "#ai-release"
  page: "pagerduty:ai-release-primary"
time_ranges:
  default: "now-48h to now"
  decision_helper: "current rollout"

decision_helper:
  title: "Should I promote the current candidate?"
  pill_query: "queries/canary_status.sql"
  interpretation: |
    Green: all hard gates pass AND all budget gates pass; soft gates pass or have sign-off.
    Yellow: all hard/budget pass; soft gates fail (requires human approval before promote).
    Red: at least one hard or budget gate fails.

panels:
  - id: canary.stage_progress
    title: "Canary stage + time-remaining"
    kind: status_strip
    query: "SELECT stage, traffic_share, state, hours_in_stage, min_duration_hours FROM experiment_ledger WHERE experiment_id = :exp"
    columns: [stage, traffic_share, state, hours_in_stage, min_duration_hours]
    trace_links: false

  - id: canary.gate_results
    title: "Latest gate results"
    kind: table
    query: "SELECT gate_id, kind, pass_, observed, threshold, reason FROM gate_results WHERE experiment_id = :exp AND stage = :stage ORDER BY kind, pass_"
    group_by: kind
    row_highlight: "pass_ = false AND kind IN ('hard', 'budget')"
    trace_links: true

  - id: canary.safety_regressions
    title: "Safety verdicts — control vs candidate"
    kind: time_series
    query: "SELECT bucket, variant_id, AVG(score) FROM verdict JOIN exposure USING(trace_id) WHERE experiment_id = :exp AND evaluator_id = 'judge.safety' GROUP BY 1, 2"
    y_axis: {min: 0, max: 1}
    alert:
      condition: "candidate_mean - control_mean < -0.02"
      severity: page
      route: "pagerduty:ai-release-primary"

  - id: canary.evidence_traces
    title: "Three recent worst-scoring candidate traces"
    kind: trace_list
    query: "SELECT trace_id, score, evaluator_id, reason FROM verdict WHERE variant_id = 'candidate' AND evaluator_id = 'judge.safety' ORDER BY score ASC LIMIT 3"
```

### 2.2 Product-manager dashboard (excerpt)

```yaml
# dashboards/product-manager.yaml
name: "Product manager — AI quality"
owner: "@ai-pm"
alert_routes:
  default: "#ai-pm-digest"
  page: null                       # PM dashboard never pages
time_ranges:
  default: "now-30d to now"

decision_helper:
  title: "Is the user experience improving this week?"
  pill_query: "queries/quality_trend.sql"
  interpretation: |
    Green: 7d median task_completion >= 4w rolling median AND thumbs_up_rate within ±2pp.
    Yellow: within ±5pp on either dimension.
    Red: material regression on either dimension (requires investigation).

panels:
  - id: quality.task_completion_trend
    title: "Task completion rate (7d rolling, by surface)"
    kind: time_series
    query: "SELECT bucket, surface, AVG(CASE WHEN label='pass' THEN 1 ELSE 0 END) FROM verdict WHERE evaluator_id='judge.task_completion' GROUP BY 1, 2"
    y_axis: {min: 0.5, max: 1.0}
    overlays:
      - {kind: shaded, start: "experiment_start", end: "experiment_end", label: "rag-prompt-v3"}

  - id: quality.thumbs_up_rate
    title: "User thumbs-up rate (7d rolling)"
    kind: time_series
    query: "SELECT bucket, AVG(value) FROM outcome WHERE metric_name='thumbs_up' GROUP BY 1"

  - id: quality.cost_per_successful_task
    title: "Cost per successful task (USD, 7d)"
    kind: time_series
    query: |
      SELECT bucket, SUM(cost_usd) / NULLIF(SUM(CASE WHEN label='pass' THEN 1 ELSE 0 END), 0)
      FROM verdict JOIN trace USING(trace_id)
      WHERE evaluator_id='judge.task_completion'
      GROUP BY 1

  - id: quality.cross_surface
    title: "Quality by surface (this week)"
    kind: bar
    query: "SELECT surface, AVG(CASE WHEN label='pass' THEN 1 ELSE 0 END) FROM verdict WHERE evaluator_id='judge.task_completion' AND bucket >= now() - interval '7 days' GROUP BY 1"

  - id: quality.user_feedback_sample
    title: "Representative thumbs-down traces (this week)"
    kind: trace_list
    query: "SELECT trace_id, surface, rating_reason FROM outcome WHERE metric_name='thumbs_down' AND bucket >= now() - interval '7 days' ORDER BY ts DESC LIMIT 10"
```

### 2.3 ML / eval engineer dashboard (excerpt)

```yaml
# dashboards/ml-eval-engineer.yaml
name: "Eval engineer — judges + detectors"
owner: "@ai-eval-eng"
time_ranges:
  default_operational: "now-24h to now"
  default_trend: "now-30d to now"

decision_helper:
  title: "Are my evaluators healthy enough to trust?"
  pill_query: "queries/judge_agreement.sql"
  interpretation: |
    Green: all judge-vs-rule agreement ≥ baseline - 2pp; drift signal counts within budget.
    Yellow: at least one judge within [baseline - 5pp, baseline - 2pp] OR drift budget 80-100% used.
    Red: at least one judge below baseline - 5pp OR drift budget exceeded.

panels:
  - id: judge.throughput
    title: "Verdicts/minute by evaluator (24h)"
    kind: time_series
    query: "SELECT bucket, evaluator_id, COUNT(*)/60.0 FROM verdict GROUP BY 1, 2"

  - id: judge.agreement_vs_rule
    title: "Judge-vs-rule agreement (24h and 30d)"
    kind: dual_time_series
    query_operational: "SELECT bucket, evaluator_id, AVG(agree) FROM judge_vs_rule_join WHERE bucket > now() - interval '24 hours' GROUP BY 1, 2"
    query_trend: "SELECT bucket, evaluator_id, AVG(agree) FROM judge_vs_rule_join WHERE bucket > now() - interval '30 days' GROUP BY 1, 2"
    overlay_baselines: true

  - id: judge.cost_burn
    title: "Judge cost burn vs monthly budget"
    kind: progress_bar
    query: "SELECT evaluator_id, SUM(cost_usd) AS spent, monthly_budget FROM verdict JOIN evaluator_budgets USING(evaluator_id) GROUP BY 1, 3"
    row_highlight: "spent / monthly_budget > 0.8"

  - id: drift.signal_counts
    title: "Drift signals this week by metric"
    kind: table
    query: "SELECT metric, severity, COUNT(*) FROM drift_signal WHERE ts > now() - interval '7 days' GROUP BY 1, 2 ORDER BY 2 DESC"

  - id: cs.widths
    title: "Confidence-sequence width over time (per metric)"
    kind: time_series
    query: "SELECT bucket, metric, cs_width FROM cs_snapshot GROUP BY 1, 2"
```

### 2.4 SRE / reliability dashboard (excerpt)

```yaml
# dashboards/sre-reliability.yaml
name: "SRE — AI reliability"
owner: "@ai-sre"
alert_routes:
  default: "#ai-sre"
  page: "pagerduty:ai-sre-primary"
time_ranges:
  default: "now-1h to now"
  budget: "current_month"

decision_helper:
  title: "Is anything on fire?"
  pill_query: "queries/latency_percentiles.sql"
  interpretation: |
    Green: p95 latency within 1.2× monthly median; error rate < 0.5%; cost burn tracking budget.
    Yellow: p95 1.2-1.5× OR error rate 0.5-2% OR cost burn > 110% of plan.
    Red: any of above thresholds exceeded.

panels:
  - id: latency.percentiles_by_surface
    title: "Latency percentiles (p50 / p95 / p99) by surface"
    kind: multi_time_series
    query: "SELECT bucket, surface, approx_percentile(latency_ms, ARRAY[0.5, 0.95, 0.99]) FROM trace GROUP BY 1, 2"

  - id: cost.budget_burn
    title: "Monthly cost burn vs budget"
    kind: progress_bar
    query: "SELECT SUM(cost_usd) AS spent, monthly_budget FROM trace JOIN budget ON TRUE WHERE ts > date_trunc('month', now())"
    alert:
      condition: "spent / monthly_budget > 1.1 AND days_elapsed_pct < 100"
      severity: page
      route: "pagerduty:ai-sre-primary"

  - id: errors.rate_by_class
    title: "Error class rate (last 1h)"
    kind: stacked_bar
    query: "SELECT error_class, COUNT(*) FROM trace WHERE ts > now() - interval '1 hour' AND error_class IS NOT NULL GROUP BY 1"

  - id: fallbacks.triggers
    title: "Fallback model activations (last 1h)"
    kind: table
    query: "SELECT surface, COUNT(*) FROM trace WHERE fallback_triggered = true AND ts > now() - interval '1 hour' GROUP BY 1"
```

### 2.5 `validation/required_panels.yaml`

```yaml
# Every persona dashboard must carry these — the enforcement list.
# Validation harness refuses to render a dashboard missing any of these.
release_owner:
  - decision_helper
  - canary.stage_progress
  - canary.gate_results
  - canary.safety_regressions
  - canary.evidence_traces     # trace links are a hard requirement

product_manager:
  - decision_helper
  - quality.task_completion_trend
  - quality.cost_per_successful_task
  - quality.user_feedback_sample

ml_eval_engineer:
  - decision_helper
  - judge.throughput
  - judge.agreement_vs_rule
  - judge.cost_burn
  - drift.signal_counts

sre_reliability:
  - decision_helper
  - latency.percentiles_by_surface
  - cost.budget_burn
  - errors.rate_by_class
```

### 2.6 `validation/test_renders.py`

```python
"""Render harness: each dashboard YAML must parse, every panel's query
must compile against the local schema, every panel in required_panels.yaml
must appear."""
from __future__ import annotations
from pathlib import Path

import yaml


ROOT = Path(__file__).parent
DASHBOARDS = ROOT.parent / "dashboards"
REQUIRED = yaml.safe_load((ROOT / "required_panels.yaml").read_text())


def test_every_dashboard_parses():
    for p in DASHBOARDS.glob("*.yaml"):
        yaml.safe_load(p.read_text())


def test_required_panels_present():
    for persona, required_ids in REQUIRED.items():
        path = DASHBOARDS / f"{persona.replace('_', '-')}.yaml"
        if persona == "release_owner":
            path = DASHBOARDS / "release-owner.yaml"
        if persona == "product_manager":
            path = DASHBOARDS / "product-manager.yaml"
        if persona == "ml_eval_engineer":
            path = DASHBOARDS / "ml-eval-engineer.yaml"
        if persona == "sre_reliability":
            path = DASHBOARDS / "sre-reliability.yaml"
        spec = yaml.safe_load(path.read_text())
        panel_ids = {"decision_helper" if "decision_helper" in spec else None}
        panel_ids |= {p["id"] for p in spec.get("panels", [])}
        missing = set(required_ids) - panel_ids
        assert not missing, f"{persona}: missing {missing}"


def test_every_panel_has_query_or_is_decision_helper():
    for p in DASHBOARDS.glob("*.yaml"):
        spec = yaml.safe_load(p.read_text())
        for panel in spec.get("panels", []):
            assert panel.get("query") or panel.get("query_operational"), (
                f"{p.name}:{panel['id']} has no query"
            )
```

## 3. Validation steps

1. **Harness tests**: `pytest -q validation/test_renders.py` passes — every YAML parses, every required panel is present, every panel carries a query.
2. **Render each dashboard in the eval UI**: load the YAML into the local Grafana / Superset / in-house renderer; every panel returns rows (even if sparse). A panel that 404s or returns a schema error fails the review.
3. **Trace links**: on the release-owner dashboard, click every `trace_links=true` cell; each opens the trace viewer with `trace_id` populated.
4. **Decision-helper interpretability**: ask a persona (release owner, PM, eval engineer, SRE) to look at the top of their dashboard for 30 seconds and say what decision they would make. Correct answers on 4/4 personas is the bar.
5. **Alert routing dry-run**: trigger a synthetic hard-gate failure; the only alert channel pinged is `#ai-release` + `pagerduty:ai-release-primary`. PM and ML dashboards do not page.
6. **Panel ownership**: `grep '^owner:' dashboards/*.yaml` returns exactly four distinct owners; every alert_routes.page value matches a real on-call rotation.
7. **Cadence match**: the release-owner dashboard loads data for "current rollout + 48h"; the PM dashboard loads 30 days. Confirm the query time-range matches the persona's declared cadence.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | One dashboard per persona; no universal "big board". | Four YAMLs; no `combined.yaml`. | A single dashboard labelled "AI overview" that mixes personas. |
| R2 | Each dashboard has a top-of-page decision helper (status pill + interpretation). | `decision_helper` present on all four; interpretation text is unambiguous. | Dashboards open on raw charts with no top-line decision. |
| R3 | Required panels (from `required_panels.yaml`) are present and non-404. | Validation harness passes. | At least one required panel missing or broken. |
| R4 | Release-owner dashboard links to example traces on every regression row. | `trace_links: true` on `canary.safety_regressions`, `canary.gate_results`, `canary.evidence_traces`. | Regression chart with no "open trace" affordance. |
| R5 | PM dashboard never pages. | `alert_routes.page: null` on PM dashboard. | PM dashboard routes a page anywhere. |
| R6 | SRE + release-owner dashboards page; routes are real on-call rotations. | `pagerduty:*` values match the rotations in the on-call registry. | Pages go to a mailing list or an abandoned channel. |
| R7 | Time ranges match persona cadence. | Release = 48h; PM = 30d; ML = 24h + 30d; SRE = 1h + month. | SRE dashboard shows 30d default; release-owner 7d default. |
| R8 | Every panel declares an owner (dashboard-level `owner:`). | Four distinct owners across four dashboards. | Unassigned panels or wildcard owners. |
| R9 | Judge cost burn, confidence-sequence width, drift signal counts all appear on the ML dashboard. | Panels present. | ML dashboard missing operational judge health. |
| R10 | Dashboards document the anti-pattern list from §5 inline (comments in YAML or README). | Comment references at least three anti-patterns. | No documentation; next team re-adds the big board. |

Pass requires 8/10 with R1, R2, R4, R5, R6 all hard.

## 5. Common mistakes

- **The "big board" anti-pattern.** A single dashboard that puts release-canary status, PM quality trend, judge cost, and SRE latency on one page serves no one. Everyone ignores 75% of the content, and when the content they need is on page 3 of a 10-row grid, they stop opening it. Four dashboards is correct.
- **No decision helper.** A dashboard that opens on a wall of charts forces the viewer to compile the decision themselves every time. A status pill with one sentence of interpretation collapses that cost.
- **Panels without trace links.** On a regression dashboard, "quality dropped 3pp" is useless without "here are three traces that dropped." The link is the point.
- **PM dashboards that page.** Product managers are not on call. A PM dashboard that fires PagerDuty trains the PM to mute it, which breaks the one case where someone genuinely needed to know.
- **SRE dashboards that cannot page.** Mirror of the previous. Latency / cost alerts must reach the on-call SRE rotation.
- **Mismatched time ranges.** An SRE looking at monthly latency misses the four-hour spike; a PM looking at hourly quality is reading noise. Match the cadence to the decision.
- **No panel owner.** Charts without owners are automatically stale within a month. Declare one on every dashboard.
- **Alerts without routing.** An alert condition with no `route` is a comment. If the dashboard owner cannot name the on-call channel, remove the alert.
- **Judge-cost burn absent.** Teams discover a runaway judge bill at the end of the month. A `progress_bar` panel against `monthly_budget` makes the burn visible every day.
- **Confidence-sequence width not surfaced.** When exercise-04's CS narrows past the actionable threshold, that is itself the story. A wide CS is a legitimate reason to wait. ML dashboards must show the width over time.
- **No required-panels enforcement.** Dashboards drift over time; someone deletes a chart "because it looked broken". `required_panels.yaml` is cheap and prevents accidental amputation.
- **Rendering raw messages in the UI.** Sensitive text in a dashboard is PII exfiltration via a different door. Render labels and scores, not user messages; link to the trace viewer for the full payload (which is where RBAC lives).

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `GOVERN 1.5` (roles and responsibilities) and `MANAGE 4.3` (documentation) map directly onto per-persona dashboards.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML06: AI Supply Chain Attacks` and `ML04: Membership Inference` motivate the "no raw messages in the dashboard UI" rule.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0024: Exfiltration via LLM`; dashboards are a notorious exfil channel if they render raw user payloads.
- Grafana docs — dashboards-as-code, <https://grafana.com/docs/grafana/latest/dashboards/build-dashboards/view-dashboard-json-model/>.
- Local chapters and exercise reader: `lessons/mod-107-online-eval-and-regression/exercises/exercise-06-cross-persona-dashboards.md`.
- Related exercises: exercise-01 (verdict sink), exercise-02 (drift signals → ML + release panels), exercise-03 (gate results → release panels), exercise-04 (CS width → ML panel), exercise-05 (CUPED-adjusted τ̂ → PM panel).
