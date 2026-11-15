# exercise-05 — CI and Online-Eval Integration — Solution

> Reference solution for `lessons/mod-110-eval-data-platform-slice/exercises/exercise-05-ci-and-online-eval-integration.md`.
> The deliverable wires the eval data plane into both the CI pipeline (mod-106) and the online-eval stream (mod-107) using the same control plane (exercise-03) and the same `eval_result` schema (exercise-01). CI gate decisions and online regression alarms both read against the same verdict table; the CI "pass" condition is a function of CI batch verdicts AND the current online SLO state.

## 1. Solution overview

Two surfaces integrate with the platform:

1. **CI**: every pull request produces a CI run (`run.kind='ci'`) against the active dataset version (exercise-02). The gate decision is a function of the run's aggregate verdicts + the online SLO state at merge time.
2. **Online**: continuous online eval (mod-107 ex-01) writes `run.kind='online'` rows. These power the drift alerts and feed the CI gate's "is the platform healthy right now?" question.

Five load-bearing decisions anchor the solution:

1. **One gate function, two inputs.** `gate(ci_run_id, now) → (decision, reasons)`. It reads CI verdicts AND the last 1h of online verdicts. Lets you block a PR that passes its own tests but ships into a degrading platform (a NIST AI RMF `MANAGE 4.1` safeguard).
2. **CI gates on distributions, not thresholds-alone.** A single pass rate ≥ 0.99 is brittle; the gate uses two tests: (a) point estimate above an absolute floor AND (b) no statistically significant drop vs. the parent `app_version`. The 2nd test catches "we're still at 98%, but we just regressed from 99.8%".
3. **The gate's answer is durable.** A gate verdict is itself written to the `eval_result`-shaped `gate_verdict` table, keyed by `(ci_run_id, gate_id, gate_version_id)`. You can replay history to see why a PR landed.
4. **Shadow mode for new judges.** A new `evaluator_version` enters CI in `shadow` for 7 days: its verdicts count toward completeness (exercise-04) but not toward the gate. After 7 days the gate owner promotes to `enforced`. Prevents a bad new judge from gating merges.
5. **Online health is a merge precondition.** If `judge_success_rate < 0.95` or `verdict_completeness < 0.99` is actively paging, the gate refuses to approve even a passing PR, with an explicit override token for emergency hotfixes. The override token is logged and audited.

```
  ┌───────────┐  POST /runs  ┌─────────────┐
  │ PR / CI   │ ───────────► │ ControlPlane│
  └───────────┘  kind=ci     └─────────────┘
       │                            │
       │                            ▼
       │                 ┌────────────────────┐
       │                 │ CIRunner (ex-03)   │
       │                 └────────────────────┘
       │                            │
       │                            ▼
       │                 ┌────────────────────┐
       └──── gate() ─── │  gate_verdict      │────► PR status
                        └────────────────────┘
                              ▲      ▲
                              │      │ reads
                        ci verdicts  online health (last 1h)
                        (eval_result)  (slo metrics)
```

## 2. Worked implementation

### 2.1 `gate/types.py`

```python
from __future__ import annotations
import uuid
from dataclasses import dataclass, field
from typing import Literal

Decision = Literal["pass", "fail", "needs_override"]


@dataclass(frozen=True)
class GateInput:
    ci_run_id: uuid.UUID
    app_version_id: str
    parent_app_version_id: str | None   # the app version on main at PR creation time
    actor: str                          # GitHub login


@dataclass(frozen=True)
class Reason:
    code: str
    message: str
    severity: Literal["info", "warning", "block"] = "block"


@dataclass(frozen=True)
class GateResult:
    ci_run_id: uuid.UUID
    decision: Decision
    reasons: tuple[Reason, ...]
    online_health_snapshot: dict[str, float | int]
    gate_version_id: str                # content hash of gate config
```

### 2.2 `gate/config.yaml` — the knobs

```yaml
gate_version_id: sha256:c1a9d...     # computed from body hash
absolute_floors:
  judge.helpfulness:   { score_mean_min: 0.80, pass_rate_min: 0.95 }
  judge.safety:        { score_mean_min: 0.95, pass_rate_min: 0.99 }
  rule.schema:         { pass_rate_min: 1.00 }

regression_test:
  enabled: true
  significance_alpha: 0.05
  min_sample_size: 50                 # if dataset is smaller, skip the test
  max_mean_drop_abs: 0.03             # even if not significant, block > 3 pp drop

shadow_evaluators:
  - evaluator_id: judge.new_rubric
    shadow_until: "2026-11-30"

online_health_preconditions:
  judge_success_rate_min: 0.95
  verdict_completeness_min: 0.99
  ingestion_freshness_max_seconds: 120
  allow_override_token: eval-gate-emergency
```

### 2.3 `gate/engine.py` — the gate function

```python
"""Pure decision logic. Reads from Postgres and the SLO exporter; writes a
gate_verdict row. No HTTP here; the caller (GitHub Action) invokes this and
posts the decision back as a PR status."""
from __future__ import annotations
import logging
import time
import uuid
from dataclasses import dataclass
from math import sqrt
from typing import Any

from .types import Decision, GateInput, GateResult, Reason

log = logging.getLogger(__name__)


GATE_VERDICT_DDL = """
CREATE TABLE IF NOT EXISTS gate_verdict (
    ci_run_id            UUID NOT NULL,
    gate_id              TEXT NOT NULL DEFAULT 'merge',
    gate_version_id      TEXT NOT NULL,
    decision             TEXT NOT NULL,                    -- pass | fail | needs_override
    reasons              JSONB NOT NULL,
    online_health        JSONB NOT NULL,
    actor                TEXT NOT NULL,
    created_ns           BIGINT NOT NULL,
    PRIMARY KEY (ci_run_id, gate_id, gate_version_id)
);
"""


class GateEngine:
    def __init__(self, conn, config: dict, slo_client, override_tokens: set[str]) -> None:
        self._conn = conn
        self._cfg = config
        self._slo = slo_client
        self._tokens = override_tokens
        self._conn.execute(GATE_VERDICT_DDL)

    def decide(self, inp: GateInput, override_token: str | None = None) -> GateResult:
        reasons: list[Reason] = []

        # 1. Online health precondition (platform-wide guard).
        health = self._online_health_snapshot()
        reasons.extend(self._check_online_health(health, override_token))

        # 2. Absolute floors per evaluator.
        reasons.extend(self._check_absolute_floors(inp.ci_run_id))

        # 3. Regression vs parent app version (two-proportion z-test).
        if inp.parent_app_version_id is not None and self._cfg["regression_test"]["enabled"]:
            reasons.extend(self._check_regression(inp))

        decision = self._decide(reasons)
        result = GateResult(
            ci_run_id=inp.ci_run_id,
            decision=decision,
            reasons=tuple(reasons),
            online_health_snapshot=health,
            gate_version_id=self._cfg["gate_version_id"],
        )
        self._persist(result, inp.actor)
        return result

    # ---- subchecks ----

    def _online_health_snapshot(self) -> dict[str, float | int]:
        snap = self._slo.read_snapshot()
        # snap shape: {"judge_success_rate": {evaluator_id: 0.993, ...},
        #              "verdict_completeness": 0.998,
        #              "ingestion_freshness_seconds": 42}
        return {
            "judge_success_rate_min_over_evaluators":
                min(snap["judge_success_rate"].values()) if snap["judge_success_rate"] else 1.0,
            "verdict_completeness": snap["verdict_completeness"],
            "ingestion_freshness_seconds": snap["ingestion_freshness_seconds"],
        }

    def _check_online_health(self, health: dict, token: str | None) -> list[Reason]:
        cfg = self._cfg["online_health_preconditions"]
        failures: list[Reason] = []
        if health["judge_success_rate_min_over_evaluators"] < cfg["judge_success_rate_min"]:
            failures.append(Reason("online.judge_success",
                f"judge success {health['judge_success_rate_min_over_evaluators']:.3f} < "
                f"min {cfg['judge_success_rate_min']}"))
        if health["verdict_completeness"] < cfg["verdict_completeness_min"]:
            failures.append(Reason("online.completeness",
                f"verdict completeness {health['verdict_completeness']:.3f} < "
                f"min {cfg['verdict_completeness_min']}"))
        if health["ingestion_freshness_seconds"] > cfg["ingestion_freshness_max_seconds"]:
            failures.append(Reason("online.freshness",
                f"ingestion lag {health['ingestion_freshness_seconds']:.0f}s > "
                f"max {cfg['ingestion_freshness_max_seconds']}s"))
        if failures and token is not None and token in self._tokens:
            # Downgrade to warnings; the override is logged in reasons.
            return [Reason(r.code, r.message + " (overridden)", "warning") for r in failures] + \
                   [Reason("override.used", f"override_token={token}", "info")]
        return failures

    def _check_absolute_floors(self, run_id: uuid.UUID) -> list[Reason]:
        failures: list[Reason] = []
        rows = self._conn.execute(
            """
            SELECT evaluator_id,
                   AVG(score)                           AS mean_score,
                   AVG((status='ok')::int)              AS pass_rate,
                   COUNT(*)                             AS n
            FROM eval_result
            WHERE run_id = %s
            GROUP BY evaluator_id
            """,
            (run_id,),
        ).fetchall()
        for evaluator_id, mean_score, pass_rate, n in rows:
            if self._is_shadow(evaluator_id):
                continue
            floor = self._cfg["absolute_floors"].get(evaluator_id, {})
            if (m := floor.get("score_mean_min")) is not None and \
               mean_score is not None and float(mean_score) < m:
                failures.append(Reason(
                    f"floor.{evaluator_id}.mean",
                    f"{evaluator_id} mean={mean_score:.3f} < floor={m}"))
            if (p := floor.get("pass_rate_min")) is not None and \
               pass_rate is not None and float(pass_rate) < p:
                failures.append(Reason(
                    f"floor.{evaluator_id}.pass_rate",
                    f"{evaluator_id} pass_rate={pass_rate:.3f} < floor={p} (n={n})"))
        return failures

    def _check_regression(self, inp: GateInput) -> list[Reason]:
        """Two-proportion z-test: PR pass rate vs parent pass rate per
        evaluator, same dataset version. Also enforce absolute drop cap."""
        failures: list[Reason] = []
        alpha = self._cfg["regression_test"]["significance_alpha"]
        min_n = self._cfg["regression_test"]["min_sample_size"]
        drop_cap = self._cfg["regression_test"]["max_mean_drop_abs"]

        pairs = self._conn.execute(
            """
            SELECT er.evaluator_id,
                   AVG(CASE WHEN t.app_version_id = %s AND er.status = 'ok' THEN 1 ELSE 0 END)
                       FILTER (WHERE t.app_version_id = %s) AS cur_rate,
                   COUNT(*) FILTER (WHERE t.app_version_id = %s) AS cur_n,
                   AVG(CASE WHEN t.app_version_id = %s AND er.status = 'ok' THEN 1 ELSE 0 END)
                       FILTER (WHERE t.app_version_id = %s) AS par_rate,
                   COUNT(*) FILTER (WHERE t.app_version_id = %s) AS par_n
            FROM eval_result er
            JOIN trace t ON t.trace_id = er.trace_id
            WHERE er.run_id IN (%s, (SELECT run_id FROM run
                                     WHERE app_version_id = %s AND kind = 'scheduled'
                                     ORDER BY started_ns DESC LIMIT 1))
            GROUP BY er.evaluator_id
            """,
            (inp.app_version_id, inp.app_version_id, inp.app_version_id,
             inp.parent_app_version_id, inp.parent_app_version_id, inp.parent_app_version_id,
             inp.ci_run_id, inp.parent_app_version_id),
        ).fetchall()

        for ev_id, cur_rate, cur_n, par_rate, par_n in pairs:
            if self._is_shadow(ev_id):
                continue
            if cur_n < min_n or par_n < min_n:
                continue
            drop = float(par_rate or 0) - float(cur_rate or 0)
            if drop > drop_cap:
                failures.append(Reason(
                    f"regression.{ev_id}.absolute",
                    f"{ev_id} dropped {drop:.3f} ({par_rate:.3f} → {cur_rate:.3f}) "
                    f"beyond {drop_cap}"))
                continue
            z = _two_proportion_z(cur_rate, cur_n, par_rate, par_n)
            if z is not None and z < -_z_for_alpha(alpha):
                failures.append(Reason(
                    f"regression.{ev_id}.significant",
                    f"{ev_id} regression z={z:.2f} at α={alpha}"))
        return failures

    def _is_shadow(self, evaluator_id: str) -> bool:
        from datetime import datetime, timezone
        for row in self._cfg["shadow_evaluators"]:
            if row["evaluator_id"] == evaluator_id:
                until = datetime.fromisoformat(row["shadow_until"]).replace(tzinfo=timezone.utc)
                return datetime.now(timezone.utc) < until
        return False

    def _decide(self, reasons: list[Reason]) -> Decision:
        if any(r.severity == "block" for r in reasons):
            # If every block-level reason is an online-health one AND overrides are allowed,
            # the override path already downgraded them to warning; we never reach here.
            return "fail"
        if any(r.code == "override.used" for r in reasons):
            return "needs_override"       # logged and visible; still merges, but flagged
        return "pass"

    def _persist(self, result: GateResult, actor: str) -> None:
        import json
        self._conn.execute(
            """
            INSERT INTO gate_verdict (ci_run_id, gate_version_id, decision,
                                      reasons, online_health, actor, created_ns)
            VALUES (%s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (ci_run_id, gate_id, gate_version_id) DO UPDATE
            SET decision = EXCLUDED.decision,
                reasons = EXCLUDED.reasons,
                online_health = EXCLUDED.online_health,
                created_ns = EXCLUDED.created_ns
            """,
            (result.ci_run_id, result.gate_version_id, result.decision,
             json.dumps([{"code": r.code, "message": r.message, "severity": r.severity}
                         for r in result.reasons]),
             json.dumps(result.online_health_snapshot), actor, time.time_ns()),
        )


def _two_proportion_z(p1: float | None, n1: int, p2: float | None, n2: int) -> float | None:
    if p1 is None or p2 is None or n1 == 0 or n2 == 0:
        return None
    p_pool = (p1 * n1 + p2 * n2) / (n1 + n2)
    denom = sqrt(p_pool * (1 - p_pool) * (1 / n1 + 1 / n2))
    if denom == 0:
        return None
    return (p1 - p2) / denom


def _z_for_alpha(alpha: float) -> float:
    # 1-sided critical value. Table lookup; keeps dependency count low.
    table = {0.10: 1.2816, 0.05: 1.6449, 0.025: 1.9600, 0.01: 2.3263, 0.005: 2.5758}
    key = min(table, key=lambda k: abs(k - alpha))
    return table[key]
```

### 2.4 `.github/workflows/eval-gate.yaml` — the CI entry point

```yaml
name: eval-gate
on:
  pull_request:
    types: [opened, synchronize, reopened]

jobs:
  run-eval:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Resolve dataset version
        run: |
          DATASET_VERSION=$(tcm active-dataset --name support.regression)
          echo "DATASET_VERSION=$DATASET_VERSION" >> "$GITHUB_ENV"

      - name: Build and pin app version
        run: |
          APP_VERSION=$(./build/resolve_app_version.sh)
          echo "APP_VERSION=$APP_VERSION" >> "$GITHUB_ENV"

      - name: Submit CI run
        id: submit
        run: |
          RUN_ID=$(curl -sf -X POST "$EVAL_PLANE/runs" \
            -H "Content-Type: application/json" \
            -H "Idempotency-Key: ${{ github.event.pull_request.head.sha }}" \
            -d '{
              "dataset_version_id":"'"$DATASET_VERSION"'",
              "evaluator_set_id":"'"$EVALUATOR_SET"'",
              "app_version_id":"'"$APP_VERSION"'",
              "runner_class":"ci",
              "priority":"fast",
              "wall_clock_budget_s": 1800,
              "cost_budget_usd": 5.0,
              "initiator":"'"${{ github.actor }}"'"
            }' | jq -r .run_id)
          echo "run_id=$RUN_ID" >> "$GITHUB_OUTPUT"

      - name: Wait for run
        run: |
          python scripts/wait_for_run.py \
            --run-id "${{ steps.submit.outputs.run_id }}" \
            --timeout-s 1800 \
            --plane "$EVAL_PLANE"

      - name: Gate decision
        id: gate
        run: |
          python -m gate.cli decide \
            --run-id "${{ steps.submit.outputs.run_id }}" \
            --app-version "$APP_VERSION" \
            --parent-app-version "$(git merge-base origin/main HEAD | xargs ./build/app_version_for.sh)" \
            --actor "${{ github.actor }}" > gate_result.json
          cat gate_result.json
          DECISION=$(jq -r .decision gate_result.json)
          echo "decision=$DECISION" >> "$GITHUB_OUTPUT"
          if [ "$DECISION" = "fail" ]; then exit 1; fi

      - name: Post gate result to PR
        if: always()
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const body = fs.existsSync('gate_result.json')
              ? '```json\n' + fs.readFileSync('gate_result.json','utf8') + '\n```'
              : 'Gate result unavailable.';
            await github.rest.issues.createComment({
              owner: context.repo.owner, repo: context.repo.repo,
              issue_number: context.issue.number,
              body: `## Eval gate — ${{ steps.gate.outputs.decision || 'unknown' }}\n${body}`,
            });
```

### 2.5 `gate/cli.py` — command-line wrapper

```python
"""Thin CLI invoked by GitHub Actions. Prints a JSON result on stdout and
exits with 0 on pass, 1 on fail, 2 on needs_override."""
from __future__ import annotations
import argparse
import json
import sys
import uuid

from .engine import GateEngine
from .types import GateInput


def main(argv: list[str] | None = None) -> int:
    p = argparse.ArgumentParser()
    p.add_argument("decide")
    p.add_argument("--run-id", required=True)
    p.add_argument("--app-version", required=True)
    p.add_argument("--parent-app-version", default=None)
    p.add_argument("--actor", required=True)
    p.add_argument("--override-token", default=None)
    args = p.parse_args(argv)

    from .wiring import build_engine
    engine = build_engine()

    result = engine.decide(
        GateInput(
            ci_run_id=uuid.UUID(args.run_id),
            app_version_id=args.app_version,
            parent_app_version_id=args.parent_app_version,
            actor=args.actor,
        ),
        override_token=args.override_token,
    )
    print(json.dumps({
        "ci_run_id": str(result.ci_run_id),
        "decision": result.decision,
        "reasons": [{"code": r.code, "message": r.message, "severity": r.severity}
                    for r in result.reasons],
        "online_health": result.online_health_snapshot,
        "gate_version_id": result.gate_version_id,
    }, indent=2))
    return {"pass": 0, "fail": 1, "needs_override": 2}[result.decision]


if __name__ == "__main__":
    sys.exit(main())
```

### 2.6 `online/join.py` — how the online loop interacts

```python
"""The online-eval loop (mod-107 ex-01) writes verdicts with `run.kind='online'`
and a long-lived `run_id` per evaluator. The gate reads the last 1h of those
verdicts through the SLO client; the two integrate without either knowing
about the other's deployment."""
from __future__ import annotations
from dataclasses import dataclass
from typing import Any


@dataclass(frozen=True)
class OnlineHealthSnapshot:
    judge_success_rate: dict[str, float]
    verdict_completeness: float
    ingestion_freshness_seconds: float


class SLOClient:
    def __init__(self, conn) -> None:
        self._conn = conn

    def read_snapshot(self) -> dict[str, Any]:
        # One query per SLO; all read the last hour.
        judge = self._conn.execute(
            """
            SELECT evaluator_id, AVG((status='ok')::int)
            FROM eval_result er
            JOIN run r ON r.run_id = er.run_id
            WHERE r.kind = 'online'
              AND er.last_updated_ns > (EXTRACT(EPOCH FROM NOW())*1e9 - 3600*1e9)
            GROUP BY evaluator_id
            """,
        ).fetchall()
        completeness = self._conn.execute(
            """
            SELECT COALESCE(MIN(frac), 1.0) FROM (
              SELECT run_id,
                     SUM((status IS NOT NULL)::int) * 1.0 / NULLIF(COUNT(*), 0) AS frac
              FROM eval_result
              WHERE last_updated_ns > (EXTRACT(EPOCH FROM NOW())*1e9 - 3600*1e9)
              GROUP BY run_id
            ) s
            """,
        ).fetchone()[0] or 1.0
        freshness = self._conn.execute(
            """
            SELECT COALESCE(
              PERCENTILE_CONT(0.95) WITHIN GROUP
                (ORDER BY (ingested_ns - started_ns) / 1e9),
              0)
            FROM trace
            WHERE ingested_ns > (EXTRACT(EPOCH FROM NOW())*1e9 - 300*1e9)
            """,
        ).fetchone()[0] or 0
        return {
            "judge_success_rate": {row[0]: float(row[1]) for row in judge},
            "verdict_completeness": float(completeness),
            "ingestion_freshness_seconds": float(freshness),
        }
```

### 2.7 `tests/test_gate_online_precondition.py`

```python
import uuid

from gate.engine import GateEngine
from gate.types import GateInput


def test_degraded_online_health_blocks_merge(seed, gate_cfg):
    engine = GateEngine(seed.conn, gate_cfg, FakeSLO(bad=True), override_tokens=set())
    result = engine.decide(GateInput(
        ci_run_id=seed.ci_run_id,
        app_version_id=seed.app_version,
        parent_app_version_id=seed.parent_app_version,
        actor="alice",
    ))
    assert result.decision == "fail"
    codes = {r.code for r in result.reasons}
    assert "online.judge_success" in codes or "online.completeness" in codes


def test_override_token_downgrades_online_block(seed, gate_cfg):
    engine = GateEngine(seed.conn, gate_cfg, FakeSLO(bad=True),
                        override_tokens={"eval-gate-emergency"})
    result = engine.decide(GateInput(
        ci_run_id=seed.ci_run_id,
        app_version_id=seed.app_version,
        parent_app_version_id=seed.parent_app_version,
        actor="alice",
    ), override_token="eval-gate-emergency")
    assert result.decision == "needs_override"
    assert any(r.code == "override.used" for r in result.reasons)


class FakeSLO:
    def __init__(self, bad: bool) -> None:
        self._bad = bad
    def read_snapshot(self) -> dict:
        if self._bad:
            return {"judge_success_rate": {"judge.helpfulness": 0.70},
                    "verdict_completeness": 0.90,
                    "ingestion_freshness_seconds": 300}
        return {"judge_success_rate": {"judge.helpfulness": 0.995},
                "verdict_completeness": 0.998,
                "ingestion_freshness_seconds": 25}
```

### 2.8 `tests/test_regression_detection.py`

```python
from gate.engine import _two_proportion_z, _z_for_alpha


def test_z_detects_meaningful_drop():
    # 0.99 → 0.80 pass rate, 200 each side.
    z = _two_proportion_z(0.80, 200, 0.99, 200)
    assert z is not None
    assert z < -_z_for_alpha(0.05)


def test_z_tolerates_noise():
    # 0.99 → 0.985 pass rate on 100 each side is noise.
    z = _two_proportion_z(0.985, 100, 0.99, 100)
    assert z is not None
    assert z > -_z_for_alpha(0.05)
```

### 2.9 `tests/test_shadow_evaluators.py`

```python
def test_shadow_evaluator_does_not_block(engine_with_shadow, seed_run_with_shadow_fail):
    result = engine_with_shadow.decide(seed_run_with_shadow_fail.input)
    assert result.decision == "pass"
    assert not any(r.code.startswith("floor.judge.new_rubric") for r in result.reasons)
```

### 2.10 Worked example — a PR reads

```
ci_run_id:        9b7b2d0c-...   (POST /runs, kind=ci, priority=fast)
dataset:          support.regression  (active)
evaluators:       rule.schema, judge.helpfulness, judge.safety, judge.new_rubric (shadow)

CI batch verdict summary:
  rule.schema          pass_rate=1.00  n=312
  judge.helpfulness    mean=0.86  pass_rate=0.97 n=312  (parent was 0.88 / 0.98)
  judge.safety         mean=0.98  pass_rate=0.99 n=312  (parent was 0.98 / 0.99)
  judge.new_rubric     mean=0.62  pass_rate=0.70 n=312  (SHADOW — ignored)

Online health @ gate time:
  judge_success_rate_min_over_evaluators: 0.996
  verdict_completeness: 0.999
  ingestion_freshness_seconds: 37

Decision: pass (0 blocks; shadow failure noted as info)
```

The same PR, with a bad helpful-judge regression:

```
Change:    helpfulness pass_rate drops 0.98 → 0.89 (n=312 vs 2000 parent)
Online:    healthy

Decision: fail
Reasons:
  - regression.judge.helpfulness.absolute: dropped 0.09 beyond 0.03
  - regression.judge.helpfulness.significant: z=-8.1 at α=0.05
```

## 3. Validation steps

1. **DDL**: load `gate_verdict` via the engine's `__init__`. The table exists after one instantiation.
2. **PR dry run**: invoke `python -m gate.cli decide --run-id <id> --app-version <sha> --parent-app-version <sha> --actor alice` against a seeded database. JSON result on stdout; exit code matches the decision.
3. **Pass path**: seed a run with pass rates above floors and no regression; gate emits `decision=pass`. `gate_verdict` row is present.
4. **Fail path**: seed a run with `judge.safety` pass rate 0.90; gate emits `decision=fail` with code `floor.judge.safety.pass_rate`.
5. **Regression path**: seed a run with `judge.helpfulness` dropping from 0.99 to 0.85; z-test fires; `decision=fail` with code `regression.judge.helpfulness.significant`.
6. **Online precondition**: force the SLO snapshot to report `judge_success_rate=0.70`; gate decision flips to `fail` even on a passing run.
7. **Override token**: pass `--override-token=eval-gate-emergency`; gate decision becomes `needs_override`; the PR status action marks the merge as "allowed with audit"; `gate_verdict.reasons` contains `override.used`.
8. **Shadow evaluator**: set `judge.new_rubric` to `shadow_until: future_date`; seed it with a pass rate of 0.50; gate ignores it; decision unaffected.
9. **GitHub Action end-to-end**: open a PR, Action runs, status check "eval-gate / pass" appears. Flip one gold answer to force a regression; next push shows "eval-gate / fail" with the reasons rendered in the PR comment.
10. **Pytest**: `pytest -q tests/`. All gate/regression/shadow/override tests pass.

## 4. Rubric / review checklist

| #   | Criterion                                                                                                   | Pass                                                                                   | Fail                                                                     |
| --- | ----------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| R1  | CI submits through the same control plane as online eval; `run.kind='ci'`.                                   | Action POSTs `/runs`; row has `kind='ci'`.                                              | CI writes directly to `eval_result`.                                     |
| R2  | Gate reads CI verdicts AND online health; a degrading platform blocks a passing PR.                          | `online_health_preconditions` enforced; test proves.                                    | Gate only looks at CI verdicts.                                           |
| R3  | Gate decision is durable in `gate_verdict`, keyed with `gate_version_id` content hash.                       | Row is persisted; config changes produce new `gate_version_id`.                         | Decision is logged only; replay impossible.                               |
| R4  | Regression test uses both absolute drop cap AND statistical significance.                                    | Both `max_mean_drop_abs` and z-test enforced; tests cover each.                          | Only one test; either overly brittle or blind to small meaningful drops.  |
| R5  | New evaluators enter in shadow and are ignored by the gate.                                                  | `shadow_evaluators` honored; test proves.                                                | New judges block merges from day one.                                     |
| R6  | Override token path is explicit, logged, and audit-visible.                                                  | `override.used` reason recorded; decision flips to `needs_override`.                    | Override is a boolean flag with no audit record.                          |
| R7  | CI run idempotent per head sha via `Idempotency-Key`.                                                        | Action header; control plane dedups; retries don't double-charge.                       | Each PR push creates a fresh run.                                        |
| R8  | Gate failure returns actionable reasons, not a single "failed".                                              | `reasons` include evaluator id, floor/parent values, z-score.                            | One line: "evaluation failed".                                            |
| R9  | Dataset version used by CI is the currently-active one (exercise-02).                                        | Action calls `tcm active-dataset`; test proves rejection if `active` list is empty.     | Hardcoded dataset version per branch.                                     |
| R10 | PR comment surfaces the gate decision and reasons; red/green status check mirrors it.                        | Workflow posts gate result as a PR comment on every push.                               | Only CI logs; reviewer has to open the Action to see why.                 |

A passing submission is 8+ of 10 with R1, R2, R3, R4, R6 all hard-required.

## 5. Common mistakes

- **Gate only reads CI verdicts.** Ships into a half-degraded platform because the PR happened to look fine on its own dataset. Read online SLO state at merge time.
- **Threshold-only pass/fail.** A single "pass_rate ≥ 0.99" check misses "we silently dropped from 0.998 to 0.991". Combine absolute floor + regression test.
- **No shadow mode.** Rolling out a new judge blocks every PR until the judge is tuned. Shadow for a week, then promote.
- **Overrides as a hidden flag.** An unaudited `--force` bit means the reason the broken PR landed is lost. Emit `override.used` as a first-class reason with the actor and token stored.
- **Non-idempotent CI submits.** GitHub re-triggers on amend / rebase; without an `Idempotency-Key` each push spins up a new run and burns cost.
- **Durable gate result missing.** A "the gate said pass yesterday but now it says fail on the same commit" conversation is unanswerable without a `gate_verdict` row. Persist it.
- **Dataset version resolved at config time, not CI time.** If the dataset is updated on main while a PR is open, the PR's gate runs against stale ground truth. Resolve the active dataset in the CI step itself.
- **Reasons as free text.** `"Evaluation failed"` isn't actionable; a code+message pair (`regression.judge.helpfulness.significant`, `z=-8.1`) points the engineer at the problem and the fix.
- **Online loop and CI write to different tables.** Doubles the maintenance and halves the visibility. Same `eval_result` shape, different `run.kind`.
- **Online health snapshot without labels.** When a block says "judge_success too low", it needs to say which judge. Snapshot is per-evaluator.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 2.11` (regression detection), `MANAGE 4.1` (post-deployment monitoring), `GOVERN 1.6` (traceability) explicitly motivate durable gate verdicts and online-feedback-blocked merges.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning) and `ML07` (transfer learning attack) motivate why CI must use the active dataset version, not a frozen copy.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.M0015` (prompt hardening) frames why a judge bump goes through shadow before enforcement: a regression in judge behavior is itself an attack surface.
- Two-proportion z-test — basic statistics; `_two_proportion_z` implementation inline above.
- GitHub Actions, <https://docs.github.com/en/actions>, for the workflow surface.
- Local exercise reader: `lessons/mod-110-eval-data-platform-slice/exercises/exercise-05-ci-and-online-eval-integration.md`.
- Related solutions: exercise-01 provides `eval_result` and `run`; exercise-02 provides the active dataset; exercise-03 provides the control plane the CI step calls; exercise-04 provides the SLO snapshot the gate reads before deciding.
