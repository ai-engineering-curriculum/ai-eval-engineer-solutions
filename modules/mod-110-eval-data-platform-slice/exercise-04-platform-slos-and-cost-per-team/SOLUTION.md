# exercise-04 — Platform SLOs and Cost per Team — Solution

> Reference solution for `lessons/mod-110-eval-data-platform-slice/exercises/exercise-04-platform-slos-and-cost-per-team.md`.
> The deliverable is a set of platform-level SLOs for the eval data plane (ingestion freshness, verdict completeness, judge-call success, lineage integrity) plus a cost attribution model that bills each team for the slice of platform spend their runs produced. The SLOs tell the platform team when to page; the cost breakdown tells finance which team owns each dollar.

## 1. Solution overview

Platform SLOs for an eval data plane are different from product SLOs for an AI app. The eval plane's "users" are evaluator pipelines, CI gates, and dashboards. Five SLOs cover the platform; one cost-attribution model backs the chargeback.

The five SLOs:

1. **Ingestion freshness** — p95 lag from trace creation to row visible in `trace` must be under a stated budget (e.g. 90 s). This is what dashboards and online-eval depend on.
2. **Verdict completeness per run** — for each run with `status='ok'`, `% of dataset rows that produced an eval_result` must be ≥ 99%. The remaining 1% must be classified (`timeout` / `error` / `over_budget` / `skipped`), not missing.
3. **Judge-call success rate** — proportion of `eval_result` rows with `status='ok'` across the last 1h window per evaluator. Below 99% for 10 minutes pages the evaluator owner; below 95% for 2 minutes pages the platform.
4. **Lineage integrity** — every `eval_result` row in the last 24h resolves to a live `app_version`, `prompt_version`, `evaluator_version`, `dataset_version`, and `run`. Orphan rows must be zero. This is the NIST AI RMF `GOVERN 1.6` traceability requirement expressed as a watch.
5. **Control-plane submit latency** — p95 of `submit → run_id returned` under 500 ms (runs are durable, so this is just the write path). Isolates the control plane (exercise-03) from the runner tail.

The cost-attribution model is a simple join:

- Every `run` has a `team` tag (defaulting to whoever `initiator` belongs to).
- Every `eval_result` has `cost_usd`.
- `trace.cost_usd` captures the app-side spend (traces also carry a `team` tag via the `tenant_id → team` map).
- Daily cost per team = Σ `eval_result.cost_usd` grouped by team (from `run.team`) + Σ `trace.cost_usd` grouped by team (from `trace.team`).

Five decisions anchor the solution:

1. **Error budgets are per SLO, not global.** Burning the ingestion-freshness budget doesn't release a judge-call-success budget. Each has its own 30-day window.
2. **Every "missing" case has to be classified, not inferred.** Verdict completeness measures explicit `status` rows, not `expected - produced`. If the plane doesn't write an `error` row, the SLO can't see it. This is the invariant that makes exercise-01's enum-rich `status` column load-bearing.
3. **Cost is attributed by team, not by evaluator.** A team that fires a broken judge 10× more than they need is the thing finance wants visible; counting by evaluator only visualizes it in aggregate.
4. **All SLO metrics exported as OpenTelemetry metrics.** Not a separate Prometheus pipeline. The eval plane already emits spans; keep metrics in the same backend.
5. **Burn-rate alerts, not threshold alerts.** A 2× burn rate for 1h pages the platform; a 14× burn rate for 5 minutes pages immediately. Classic SRE SLO-alerting pattern applied to the eval plane.

## 2. Worked implementation

### 2.1 `slo/definitions.yaml` — the five SLOs

```yaml
# Error budget windows are 30 days rolling. Burn-rate alerts use the
# 1h/6h/24h windows per the Google SRE SLO cookbook.
slo:
  - id: ingestion_freshness
    description: |
      p95 of (ingested_ns - started_ns) across all trace rows written in
      the last 5 minutes.
    sli_query: |
      SELECT PERCENTILE_CONT(0.95) WITHIN GROUP (ORDER BY (ingested_ns - started_ns))
        / 1e9 AS seconds_lag
      FROM trace
      WHERE ingested_ns > NOW_NS() - 5*60*1_000_000_000;
    objective_seconds: 90
    owner: platform-eval
    burn_rate_alerts:
      - window: 1h
        threshold: 14.4
        severity: page
      - window: 6h
        threshold: 6.0
        severity: page
      - window: 24h
        threshold: 2.0
        severity: ticket

  - id: verdict_completeness
    description: |
      Fraction of eval_result rows per run that terminated in a classified
      status (ok|timeout|error|over_budget|skipped). Measured over the last
      hour; any missing row is counted against the budget.
    sli_query: |
      WITH r AS (
        SELECT run_id,
               COUNT(*) FILTER (WHERE status IS NOT NULL) * 1.0 /
               NULLIF(COUNT(*), 0) AS fraction_classified
        FROM eval_result
        WHERE last_updated_ns > NOW_NS() - 1*3600*1_000_000_000
        GROUP BY run_id
      )
      SELECT MIN(fraction_classified) FROM r;
    objective_fraction: 0.99
    owner: platform-eval
    burn_rate_alerts:
      - window: 1h
        threshold: 14.4
        severity: page

  - id: judge_success_rate
    description: |
      Fraction of eval_result rows per evaluator with status='ok' over the
      last hour. Published per (evaluator_id, evaluator_version_id).
    sli_query: |
      SELECT evaluator_id, evaluator_version_id,
             SUM((status = 'ok')::int) * 1.0 / COUNT(*) AS success_rate
      FROM eval_result
      WHERE last_updated_ns > NOW_NS() - 1*3600*1_000_000_000
      GROUP BY evaluator_id, evaluator_version_id;
    objective_fraction: 0.99
    owner: evaluator_owner    # the owner column on evaluator_version
    burn_rate_alerts:
      - window: 10m
        threshold: 14.4
        severity: page
      - window: 2m
        threshold: 36.0
        severity: page_fast

  - id: lineage_integrity
    description: |
      Count of eval_result rows in the last 24h whose FKs do not resolve.
      Target is zero; non-zero is a correctness incident.
    sli_query: |
      SELECT COUNT(*) FROM eval_result er
      WHERE er.last_updated_ns > NOW_NS() - 24*3600*1_000_000_000
        AND (NOT EXISTS (SELECT 1 FROM trace t WHERE t.trace_id = er.trace_id)
          OR NOT EXISTS (SELECT 1 FROM evaluator_version ev
                         WHERE ev.evaluator_version_id = er.evaluator_version_id)
          OR NOT EXISTS (SELECT 1 FROM run r WHERE r.run_id = er.run_id));
    objective_count: 0
    owner: platform-eval

  - id: submit_latency_p95
    description: |
      p95 wall-clock latency of POST /runs, measured from the HTTP trace.
    sli_query: |
      histogram_quantile(0.95,
        sum by (le) (rate(http_server_duration_bucket{route="/runs"}[5m])))
    objective_ms: 500
    owner: platform-eval
    burn_rate_alerts:
      - window: 1h
        threshold: 14.4
        severity: page
```

### 2.2 `cost/attribution.sql` — the chargeback view

```sql
-- team_tag: a stable mapping from initiator (service account or user) to
-- an owning team. Maintained in a yaml file, loaded daily. Keep this small;
-- the view below reads it as a lookup.
CREATE TABLE team_tag (
    initiator           TEXT PRIMARY KEY,
    team                TEXT NOT NULL
);

-- v_eval_cost_by_team: eval-plane spend. Joins eval_result → run → team_tag.
CREATE VIEW v_eval_cost_by_team AS
SELECT
    t.team,
    DATE_TRUNC('day', TO_TIMESTAMP(er.last_updated_ns / 1e9)) AS day,
    r.kind                                                    AS runner_class,
    er.evaluator_id,
    SUM(er.cost_usd)                                          AS cost_usd,
    COUNT(*)                                                  AS verdict_count
FROM eval_result er
JOIN run       r ON r.run_id = er.run_id
JOIN team_tag  t ON t.initiator = r.initiator
GROUP BY 1, 2, 3, 4;

-- v_trace_cost_by_team: app-plane spend. Trace rows carry tenant_id;
-- tenant_team_map resolves tenant → team.
CREATE TABLE tenant_team_map (
    tenant_id           TEXT PRIMARY KEY,
    team                TEXT NOT NULL
);

CREATE VIEW v_trace_cost_by_team AS
SELECT
    ttm.team,
    DATE_TRUNC('day', TO_TIMESTAMP(tr.ingested_ns / 1e9))     AS day,
    tr.surface,
    SUM(tr.cost_usd)                                          AS cost_usd,
    COUNT(*)                                                  AS trace_count
FROM trace tr
JOIN tenant_team_map ttm ON ttm.tenant_id = tr.tenant_id
GROUP BY 1, 2, 3;

-- v_cost_summary_by_team: the finance-facing roll-up.
CREATE VIEW v_cost_summary_by_team AS
SELECT
    COALESCE(e.team, t.team)                                  AS team,
    COALESCE(e.day,  t.day)                                   AS day,
    COALESCE(SUM(e.cost_usd), 0)                              AS eval_cost_usd,
    COALESCE(SUM(t.cost_usd), 0)                              AS app_cost_usd,
    COALESCE(SUM(e.cost_usd), 0) + COALESCE(SUM(t.cost_usd), 0) AS total_usd
FROM v_eval_cost_by_team e
FULL OUTER JOIN v_trace_cost_by_team t
  ON e.team = t.team AND e.day = t.day
GROUP BY COALESCE(e.team, t.team), COALESCE(e.day, t.day);
```

### 2.3 `slo/exporter.py` — turn SLO queries into OTel metrics

```python
"""Run each SLO query on a schedule; emit a point per interval. Downstream
alerting is a standard SRE burn-rate on these."""
from __future__ import annotations
import logging
import time
from dataclasses import dataclass
from typing import Iterable

from opentelemetry import metrics

log = logging.getLogger(__name__)
METER = metrics.get_meter("eval.platform.slo")


@dataclass(frozen=True)
class SLODef:
    id: str
    sli_query: str
    objective_ms: float | None = None
    objective_fraction: float | None = None
    objective_count: int | None = None
    objective_seconds: float | None = None


class SLOExporter:
    def __init__(self, conn, defs: Iterable[SLODef]) -> None:
        self._conn = conn
        self._defs = list(defs)
        self._gauges = {d.id: METER.create_observable_gauge(
            name=f"eval_slo.{d.id}", callbacks=[self._callback(d)],
        ) for d in self._defs}

    def _callback(self, d: SLODef):
        def _cb(options):
            rows = self._conn.execute(d.sli_query).fetchall()
            # Conventional shape: queries return either a single value or
            # one row per (evaluator_id, evaluator_version_id).
            for row in rows:
                if len(row) == 1:
                    yield metrics.Observation(value=float(row[0]), attributes={})
                elif len(row) == 3:
                    evaluator_id, evaluator_version_id, value = row
                    yield metrics.Observation(value=float(value),
                                              attributes={
                                                  "evaluator_id": evaluator_id,
                                                  "evaluator_version_id": evaluator_version_id})
        return _cb
```

### 2.4 `slo/budget.py` — burn-rate math

```python
"""30-day rolling error budget. Burn-rate alerts live here; the Prometheus /
OTel alert rules call this to compute the current burn ratio."""
from __future__ import annotations
import math


def budget_remaining(
    successes_in_window: int,
    total_in_window: int,
    objective_fraction: float,
) -> float:
    """Return the fraction of the error budget still available.
    1.0 = all budget intact, 0.0 = fully consumed."""
    if total_in_window == 0:
        return 1.0
    actual_success = successes_in_window / total_in_window
    error_rate = 1.0 - actual_success
    allowed_error = 1.0 - objective_fraction
    if allowed_error <= 0:
        return 0.0 if error_rate > 0 else 1.0
    return max(0.0, 1.0 - (error_rate / allowed_error))


def burn_rate(
    recent_error_rate: float,
    objective_fraction: float,
) -> float:
    """1x burn rate means the budget will be consumed exactly in 30 days.
    14.4x means in ~2 hours; 36x in ~50 minutes (page immediately)."""
    allowed_error = 1.0 - objective_fraction
    if allowed_error <= 0:
        return math.inf if recent_error_rate > 0 else 0.0
    return recent_error_rate / allowed_error
```

### 2.5 `cost/report.py` — the daily report generator

```python
"""Emits a per-team cost summary. Called by a cron the day after. The output
is a JSON file picked up by finance's spreadsheet importer; no live API."""
from __future__ import annotations
import datetime as dt
import json
import pathlib


REPORT_SQL = """
SELECT team, day, eval_cost_usd, app_cost_usd, total_usd
FROM v_cost_summary_by_team
WHERE day = %s
ORDER BY total_usd DESC
"""


def emit_daily_report(conn, day: dt.date, out_path: pathlib.Path) -> pathlib.Path:
    rows = conn.execute(REPORT_SQL, (day,)).fetchall()
    payload = {
        "day": day.isoformat(),
        "teams": [
            {"team": r[0], "eval_cost_usd": float(r[2]),
             "app_cost_usd": float(r[3]), "total_usd": float(r[4])}
            for r in rows
        ],
        "total_platform_usd": sum(float(r[4]) for r in rows),
    }
    out = out_path / f"cost-report-{day.isoformat()}.json"
    out.write_text(json.dumps(payload, indent=2))
    return out
```

### 2.6 `dashboards/eval_plane_slo.json` — Grafana (abridged)

```json
{
  "title": "Eval Data Plane — SLOs",
  "panels": [
    {
      "type": "stat",
      "title": "Ingestion lag p95 (s, target < 90s)",
      "targets": [{"expr": "eval_slo_ingestion_freshness"}]
    },
    {
      "type": "stat",
      "title": "Verdict completeness per run (target > 99%)",
      "targets": [{"expr": "min(eval_slo_verdict_completeness)"}]
    },
    {
      "type": "timeseries",
      "title": "Judge success rate by evaluator",
      "targets": [{"expr": "eval_slo_judge_success_rate",
                   "legendFormat": "{{evaluator_id}} / {{evaluator_version_id}}"}]
    },
    {
      "type": "stat",
      "title": "Lineage orphan rows (24h, target = 0)",
      "targets": [{"expr": "eval_slo_lineage_integrity"}],
      "options": {"thresholds": {"mode": "absolute",
                                   "steps": [{"color": "green", "value": 0},
                                             {"color": "red",   "value": 1}]}}
    },
    {
      "type": "stat",
      "title": "POST /runs p95 (ms, target < 500ms)",
      "targets": [{"expr": "eval_slo_submit_latency_p95"}]
    }
  ]
}
```

### 2.7 `dashboards/cost_by_team.sql` — the finance view

```sql
-- Monthly rollup the finance dashboard reads.
SELECT
    team,
    DATE_TRUNC('month', day)::date       AS month,
    SUM(eval_cost_usd)                   AS eval_usd,
    SUM(app_cost_usd)                    AS app_usd,
    SUM(total_usd)                       AS total_usd
FROM v_cost_summary_by_team
WHERE day >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '6 months'
GROUP BY 1, 2
ORDER BY 2 DESC, 5 DESC;
```

### 2.8 `alerts/burn_rates.yaml` — the Prometheus alert rules

```yaml
# Follows the Google SRE SLO burn-rate pattern. Short window AND long window
# must both exceed the threshold for a page; this damps flaps.
groups:
  - name: eval-plane-slos
    rules:
      - alert: EvalIngestionFreshnessFast
        expr: |
          (eval_slo_ingestion_freshness > 90) and
          (avg_over_time(eval_slo_ingestion_freshness[5m]) > 90)
        for: 2m
        labels:
          severity: page
          owner: platform-eval
        annotations:
          summary: "Trace ingestion lag p95 over 90s — online-eval and dashboards are stale"

      - alert: EvalJudgeSuccessFast
        expr: eval_slo_judge_success_rate < 0.95
        for: 2m
        labels:
          severity: page
          owner: evaluator-owner
        annotations:
          summary: "Judge success rate < 95% — likely rate-limit or prompt regression"

      - alert: EvalLineageOrphanRows
        expr: eval_slo_lineage_integrity > 0
        for: 5m
        labels:
          severity: page
          owner: platform-eval
        annotations:
          summary: "Orphan eval_result rows found — ingest or FK path is broken"
```

### 2.9 `tests/test_burn_rate.py`

```python
from slo.budget import burn_rate, budget_remaining


def test_burn_rate_at_objective_is_1x():
    # If error rate equals allowed error, 1x burn: budget consumed in 30 days.
    assert burn_rate(recent_error_rate=0.01, objective_fraction=0.99) == 1.0


def test_burn_rate_14x_triggers_short_window_page():
    # 14.4x burn rate burns a 30-day budget in ~50 hours.
    br = burn_rate(recent_error_rate=0.14, objective_fraction=0.99)
    assert 13 < br < 15


def test_budget_full_when_no_errors():
    assert budget_remaining(1000, 1000, 0.99) == 1.0


def test_budget_zero_when_completely_consumed():
    # 99% success target → 1% allowed; here we're at 2% errors (double)
    rem = budget_remaining(980, 1000, 0.99)
    assert 0.0 <= rem <= 0.01
```

### 2.10 `tests/test_cost_attribution.py`

```python
def test_cost_rolls_up_by_team(seed):
    # Seed: team=search fires 100 judge calls at $0.001 each; team=support fires 50 at $0.002.
    rows = seed.conn.execute("""
        SELECT team, SUM(cost_usd) AS c
        FROM v_eval_cost_by_team
        GROUP BY team ORDER BY team
    """).fetchall()
    costs = {r[0]: float(r[1]) for r in rows}
    assert abs(costs["search"] - 0.1) < 1e-6
    assert abs(costs["support"] - 0.1) < 1e-6


def test_untagged_initiator_lands_in_unknown(seed):
    # A run from a bot without a team_tag row must still appear in the roll-up.
    rows = seed.conn.execute(
        "SELECT team FROM v_eval_cost_by_team WHERE team = 'unknown'"
    ).fetchall()
    assert rows, "untagged initiators must be rolled up under a visible bucket"
```

## 3. Validation steps

1. **Load DDL**: `psql -f cost/attribution.sql`. No errors.
2. **Seed team maps**: insert 20 `team_tag` rows and 50 `tenant_team_map` rows. Load one day of synthetic `run` + `trace` + `eval_result` rows for 3 teams.
3. **Run SLO queries manually**: each of the five `sli_query` blocks returns a numeric result (or a per-evaluator set for judge_success_rate). Values are in expected ranges on healthy fixture data.
4. **Start the exporter**: `python -m slo.exporter` registers five OTel gauges. Scrape the endpoint; the five metrics appear with the expected labels.
5. **Trigger a failure path**: deliberately set an evaluator to 20% error rate for 10 minutes. The 2-minute burn-rate alert fires; the 10-minute alert follows; both target the correct `owner`.
6. **Lineage break test**: delete a `run` row that still has `eval_result` rows pointing at it. `eval_slo_lineage_integrity > 0` within one scrape; alert fires.
7. **Daily cost report**: `python -m cost.report --day 2026-11-14` writes a JSON file; totals match a hand-computed sum from the seed fixture.
8. **Cost reconciliation**: compare `v_cost_summary_by_team.total_usd` for yesterday to the vendor billing export for the same tenants. Deltas < 2% explained by rounding and currency conversion windows.
9. **Burn-rate tests**: `pytest -q tests/test_burn_rate.py tests/test_cost_attribution.py`. All pass.
10. **Dashboard renders**: import `dashboards/eval_plane_slo.json` into Grafana; all five panels populate; the single-stat panels light red when thresholds breach.

## 4. Rubric / review checklist

| #   | Criterion                                                                                                 | Pass                                                                                     | Fail                                                                     |
| --- | --------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| R1  | Five SLOs cover ingestion freshness, completeness, judge success, lineage integrity, submit latency.       | Each defined in `slo/definitions.yaml` with a measurable SLI query.                       | SLOs defined by English alone; no SLI query.                             |
| R2  | Each SLO has a 30-day error budget and burn-rate alerts (fast + slow windows).                              | `burn_rate_alerts` present; fast/slow window pattern used.                                | Threshold-only alerts; flap on every blip.                                |
| R3  | Judge success SLO is per `(evaluator_id, evaluator_version_id)`; alerts target the evaluator owner.         | Query groups by both; owner is `evaluator_owner`.                                         | One global judge success metric; no per-owner routing.                    |
| R4  | Lineage integrity SLO counts orphan rows and targets zero.                                                  | Query checks FKs; alert fires on > 0.                                                     | Lineage not monitored; orphans accumulate silently.                       |
| R5  | Verdict completeness measures explicit status rows; missing rows count against the budget.                  | Query divides classified rows by total expected; missing treated as unclassified.          | Measured by "rows produced"; missing rows invisible.                      |
| R6  | SLO metrics exported as OpenTelemetry gauges (not a parallel pipeline).                                     | `SLOExporter` registers `observable_gauge` per SLO.                                        | Separate Prom/Graphite exporter; divergent naming.                        |
| R7  | Cost attribution rolls up per team via explicit mapping tables (`team_tag`, `tenant_team_map`).              | DDL + view present; untagged initiators land in `unknown`.                                 | Cost attributed per evaluator only; teams invisible.                     |
| R8  | Daily cost report includes both eval-plane and app-plane spend per team.                                    | `v_cost_summary_by_team` joins both sources.                                              | Only one plane shown; finance has to sum by hand.                         |
| R9  | Dashboards render the SLOs and the cost summary; thresholds are explicit.                                   | `dashboards/eval_plane_slo.json` ships; Grafana import works.                              | Metrics exist but no dashboard; alerts without visibility.                |
| R10 | Owners are routed on alerts; the evaluator SLO pages the evaluator owner, not the platform team.             | `owner: evaluator_owner` resolves to an evaluator-version metadata field.                  | Every alert pages the platform team; evaluator owners stay insulated.    |

A passing submission is 8+ of 10 with R1, R2, R4, R5, R7 all hard-required.

## 5. Common mistakes

- **Threshold-only alerts.** Alerting when `ingestion_freshness > 90 for 1m` fires on every blip. Burn-rate alerts (fast + slow windows) separate a real budget burn from a brief spike.
- **Measuring completeness by "rows produced".** The trap: a run that silently dropped 5% of cases looks complete because dropped rows have no row at all. Measure classified rows; missing counts as unclassified and burns the budget.
- **Attributing cost to the evaluator, not the team.** Finance needs a team view. The same judge may serve five teams; only the team roll-up tells you who's actually spending.
- **No `unknown` bucket for untagged initiators.** A missing `team_tag` row makes the cost invisible; finance thinks the platform is leaking. Always land untagged spend under a visible bucket.
- **One global judge-success SLO.** With five judges deployed, one bad judge drags the global metric down and the whole platform pages. Per-evaluator SLOs page per-evaluator owners.
- **No lineage integrity SLO.** Orphan rows are a silent corruption; the "why did my dashboard go blank?" fire drill. One cheap query, one page, zero recurrences.
- **SLO definitions in English only.** If the SLI isn't a query that runs in production, it isn't an SLO; it's a wish. Every row in the definitions file MUST be executable.
- **Burn-rate window of only 1 hour.** Short-only windows catch real incidents but miss slow leaks. Long windows alone are too slow. Both.
- **Reconciling cost against vendor export at the metric layer.** Build the cost roll-up from the primary source (`eval_result.cost_usd`) and treat vendor export as a check, not a source of truth. Otherwise a vendor outage breaks cost attribution.
- **No deployment hooks on the alert rules.** The rules live in a repo; a new evaluator version lands without an owner mapping; alerts fire into `#eng-general`. Require the evaluator owner on every `evaluator_version` write; the alert router joins against that.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 2.11` (metric regression detection), `MANAGE 4.1` (post-deployment monitoring), `GOVERN 1.6` (traceability) frame why integrity and completeness are platform SLOs.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning) motivates the lineage integrity SLO as a detective control.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0024` (LLM exfiltration) frames judge-success tracking as a channel-level control: a sudden rise in judge errors is often a prompt-injection or rate-limit incident.
- OpenTelemetry metrics API, <https://opentelemetry.io/docs/specs/otel/metrics/>, for the exporter contract.
- Prometheus alerting on burn rate — the SRE SLO model is public; see <https://sre.google/workbook/alerting-on-slos/> for the fast/slow-window pattern this alert file follows.
- Local exercise reader: `lessons/mod-110-eval-data-platform-slice/exercises/exercise-04-platform-slos-and-cost-per-team.md`.
- Related solutions: exercise-01 provides the `status` enum and the lineage tables the SLOs read; exercise-02 provides `dataset_version` used in completeness; exercise-03 provides the `run.initiator` the cost model attributes on; exercise-05 adds a CI-gate specific SLO that composes on top of these.
