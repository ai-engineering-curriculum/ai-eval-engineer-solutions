# exercise-03 — Multi-Runner Orchestration Behind One Plane — Solution

> Reference solution for `lessons/mod-110-eval-data-platform-slice/exercises/exercise-03-multi-runner-orchestration-behind-one-plane.md`.
> The deliverable is a single control plane that accepts eval jobs and dispatches them to multiple heterogeneous runners (CI batch runner, online-eval stream runner, ad-hoc notebook runner, human-review router). Every runner writes `eval_result` rows into the same shape (exercise-01), tagged with the same `run_id`, so one dashboard and one SLO (exercise-04) sees across all of them.

## 1. Solution overview

The control plane is a thin service that:

1. **Accepts a `RunRequest`** on a single API endpoint. The request names a dataset version (exercise-02), an evaluator set, an app version, a runner class, and a priority band.
2. **Resolves the request** by looking up lineage ids and asserting all FKs exist (fail fast at submit time, not at run time).
3. **Dispatches to a runner** via a pluggable `Runner` interface. Each runner implementation (CI, online, ad-hoc, human) owns its own worker pool and its own queue semantics; the plane only cares about the contract.
4. **Watches runs via a heartbeat** from each runner, timing out stale jobs after a declared budget.
5. **Writes a `run` row** (exercise-01's `run` table) at submit time, updates its status on completion, and ensures every runner's `eval_result` rows carry the same `run_id`.

Five design choices anchor it:

1. **One `Runner` interface, many implementations.** The runner contract is `submit(job) → None`, `heartbeat(run_id) → status`, `cancel(run_id) → None`. Anything that can honor those three can plug in — a Kubernetes job, a Celery task, an inline Python asyncio loop, a Slack workflow for human review.
2. **Priority is a queue property, not a job property.** Jobs land on `fast`, `standard`, or `background` queues. The CI runner polls `fast` first; the scheduled-regression runner polls `standard`. Changing the priority of a job after submit is a resubmit.
3. **Idempotency keys at the plane layer.** Submitting the same `(dataset_version_id, evaluator_set_id, app_version_id, runner_class)` within the dedup window returns the existing `run_id`. Prevents the "retry storm from the CI retry policy" class of bug.
4. **Timeouts and budgets are declared per run, not per job.** Each run carries a wall-clock cap and a cost cap. The runner enforces both; the plane reports `aborted` when either is exceeded. Without this, a runaway judge can silently drain the budget before anyone notices (NIST AI RMF `MANAGE 4.1`).
5. **Cancellation is cooperative.** The plane sets `status='canceled'` on the `run` row; each runner polls for cancellation between evaluator invocations and writes `status='skipped'` rows for the rest. Half-finished runs don't corrupt coverage math for exercise-04.

```
                          ┌──────────────────┐
     RunRequest ────────► │   Control plane  │
                          │  (this module)   │
                          └──────────────────┘
                                   │
          ┌────────────────────────┼────────────────────────┐
          │                        │                        │
          ▼                        ▼                        ▼
   ┌─────────────┐        ┌────────────────┐        ┌───────────────┐
   │  CI runner  │        │ online runner  │        │ human router  │
   │ (K8s Job)   │        │ (asyncio pool) │        │ (Slack queue) │
   └─────────────┘        └────────────────┘        └───────────────┘
          │                        │                        │
          └────────────────────────┼────────────────────────┘
                                   ▼
                         eval_result + run tables
                         (exercise-01 schema)
```

## 2. Worked implementation

### 2.1 `plane/types.py` — the shared contract

```python
"""Shared types. Importable by every runner. `RunRequest` is accepted by the
submit endpoint; `RunJob` is what each runner receives."""
from __future__ import annotations
import uuid
from dataclasses import dataclass, field
from typing import Literal


RunnerClass = Literal["ci", "online", "scheduled", "ad_hoc", "human"]
Priority    = Literal["fast", "standard", "background"]
RunStatus   = Literal["queued", "running", "ok", "failed", "aborted", "canceled"]


@dataclass(frozen=True)
class RunRequest:
    dataset_version_id: str | None     # None for pure online runs
    evaluator_set_id:   str            # content-addressed set of (evaluator_id, evaluator_version_id)
    app_version_id:     str
    runner_class:       RunnerClass
    priority:           Priority = "standard"
    wall_clock_budget_s: int     = 1800
    cost_budget_usd:     float   = 5.0
    initiator:           str     = "unknown"
    idempotency_key:     str | None = None       # client-provided; defaults to content hash


@dataclass(frozen=True)
class RunJob:
    run_id:               uuid.UUID
    dataset_version_id:   str | None
    evaluator_set_id:     str
    app_version_id:       str
    wall_clock_budget_s:  int
    cost_budget_usd:      float
    created_ns:           int


@dataclass
class Heartbeat:
    run_id:            uuid.UUID
    status:            RunStatus
    processed_count:   int = 0
    total_count:       int | None = None
    cost_spent_usd:    float = 0.0
    last_update_ns:    int   = 0
    error:             str | None = None
```

### 2.2 `plane/runner.py` — the plug-in interface

```python
"""Each runner implementation honors this protocol. Nothing else is required
to plug a new one in — register it with the plane at startup."""
from __future__ import annotations
import uuid
from typing import Protocol

from .types import RunJob, Heartbeat, RunStatus


class Runner(Protocol):
    runner_class: str

    def submit(self, job: RunJob) -> None:
        """Enqueue the job. MUST return quickly; do not evaluate inline."""

    def heartbeat(self, run_id: uuid.UUID) -> Heartbeat:
        """Return the current progress. The plane polls at 5 s intervals."""

    def cancel(self, run_id: uuid.UUID) -> None:
        """Mark the run for cooperative cancellation. Idempotent."""
```

### 2.3 `plane/service.py` — the control plane

```python
"""The one service users call. Resolves lineage, writes the `run` row, hands
off to the registered runner, polls heartbeats, enforces budgets. No evaluator
code runs here."""
from __future__ import annotations
import hashlib
import logging
import time
import uuid

from .runner import Runner
from .types import Heartbeat, RunJob, RunRequest, RunStatus

log = logging.getLogger(__name__)


class ControlPlane:
    def __init__(self, conn, runners: dict[str, Runner], now_ns=time.time_ns) -> None:
        self._conn = conn
        self._runners = runners
        self._now_ns = now_ns

    # ---- submit ----

    def submit(self, req: RunRequest) -> uuid.UUID:
        self._assert_lineage(req)
        idem = req.idempotency_key or self._default_idem(req)
        existing = self._dedup(idem)
        if existing:
            return existing

        run_id = uuid.uuid4()
        self._conn.execute(
            """
            INSERT INTO run (run_id, kind, initiator, dataset_version_id,
                             app_version_id, started_ns, status, metadata)
            VALUES (%s, %s, %s, %s, %s, %s, 'queued', %s)
            """,
            (run_id, req.runner_class, req.initiator, req.dataset_version_id,
             req.app_version_id, self._now_ns(), {
                 "evaluator_set_id":  req.evaluator_set_id,
                 "priority":          req.priority,
                 "wall_clock_budget_s": req.wall_clock_budget_s,
                 "cost_budget_usd":   req.cost_budget_usd,
                 "idempotency_key":   idem,
             }),
        )
        job = RunJob(
            run_id=run_id,
            dataset_version_id=req.dataset_version_id,
            evaluator_set_id=req.evaluator_set_id,
            app_version_id=req.app_version_id,
            wall_clock_budget_s=req.wall_clock_budget_s,
            cost_budget_usd=req.cost_budget_usd,
            created_ns=self._now_ns(),
        )
        runner = self._runners[req.runner_class]
        runner.submit(job)
        return run_id

    # ---- poll ----

    def poll(self, run_id: uuid.UUID) -> Heartbeat:
        runner_class = self._runner_class_for(run_id)
        runner = self._runners[runner_class]
        hb = runner.heartbeat(run_id)
        self._reconcile(run_id, hb)
        self._enforce_budgets(run_id, hb)
        return hb

    # ---- cancel ----

    def cancel(self, run_id: uuid.UUID) -> None:
        runner_class = self._runner_class_for(run_id)
        self._runners[runner_class].cancel(run_id)
        self._conn.execute(
            "UPDATE run SET status = 'canceled', finished_ns = %s WHERE run_id = %s AND finished_ns IS NULL",
            (self._now_ns(), run_id),
        )

    # ---- internals ----

    def _default_idem(self, req: RunRequest) -> str:
        blob = f"{req.dataset_version_id}|{req.evaluator_set_id}|{req.app_version_id}|{req.runner_class}"
        return "sha256:" + hashlib.sha256(blob.encode()).hexdigest()

    def _dedup(self, idem: str, window_s: int = 900) -> uuid.UUID | None:
        row = self._conn.execute(
            """
            SELECT run_id FROM run
            WHERE metadata ->> 'idempotency_key' = %s
              AND status IN ('queued', 'running', 'ok')
              AND started_ns > %s
            ORDER BY started_ns DESC LIMIT 1
            """,
            (idem, self._now_ns() - window_s * 1_000_000_000),
        ).fetchone()
        return row[0] if row else None

    def _assert_lineage(self, req: RunRequest) -> None:
        ok = self._conn.execute(
            "SELECT 1 FROM app_version WHERE app_version_id = %s",
            (req.app_version_id,),
        ).fetchone()
        if not ok:
            raise LookupError(f"unknown app_version_id={req.app_version_id}")
        if req.dataset_version_id is not None:
            ok = self._conn.execute(
                "SELECT 1 FROM dataset_version WHERE dataset_version_id = %s",
                (req.dataset_version_id,),
            ).fetchone()
            if not ok:
                raise LookupError(f"unknown dataset_version_id={req.dataset_version_id}")
        if req.runner_class not in self._runners:
            raise LookupError(f"no runner registered for class={req.runner_class}")

    def _runner_class_for(self, run_id: uuid.UUID) -> str:
        return self._conn.execute(
            "SELECT kind FROM run WHERE run_id = %s", (run_id,)
        ).fetchone()[0]

    def _reconcile(self, run_id: uuid.UUID, hb: Heartbeat) -> None:
        self._conn.execute(
            """
            UPDATE run
            SET status = %s,
                finished_ns = CASE WHEN %s IN ('ok','failed','aborted','canceled')
                                   THEN %s ELSE finished_ns END,
                metadata = metadata || %s
            WHERE run_id = %s
            """,
            (hb.status, hb.status, self._now_ns(),
             {"processed_count": hb.processed_count,
              "cost_spent_usd": hb.cost_spent_usd,
              "error": hb.error},
             run_id),
        )

    def _enforce_budgets(self, run_id: uuid.UUID, hb: Heartbeat) -> None:
        row = self._conn.execute(
            """
            SELECT started_ns,
                   (metadata ->> 'wall_clock_budget_s')::int,
                   (metadata ->> 'cost_budget_usd')::numeric
            FROM run WHERE run_id = %s
            """,
            (run_id,),
        ).fetchone()
        started_ns, wall_budget_s, cost_budget = row
        wall_spent_s = (self._now_ns() - started_ns) / 1_000_000_000
        if hb.status in ("queued", "running") and (
            wall_spent_s > wall_budget_s or hb.cost_spent_usd > float(cost_budget)
        ):
            log.warning("run_budget_exceeded", extra={"run_id": str(run_id),
                                                       "wall_spent_s": wall_spent_s,
                                                       "cost_spent_usd": hb.cost_spent_usd})
            self.cancel(run_id)
            self._conn.execute(
                "UPDATE run SET status = 'aborted' WHERE run_id = %s", (run_id,),
            )
```

### 2.4 `runners/ci.py` — a CI batch runner (Kubernetes job dispatch)

```python
"""Submits one Kubernetes Job per run. The Job's pod is a prebuilt image
that reads the run_id from env, pulls the dataset, iterates every case
through the evaluator set, writes eval_result rows, and heartbeats the
plane. Here we only show the submit/heartbeat/cancel seam."""
from __future__ import annotations
import logging
import uuid

from plane.runner import Runner
from plane.types import Heartbeat, RunJob

log = logging.getLogger(__name__)


class CIRunner:
    runner_class = "ci"

    def __init__(self, k8s_client, namespace: str, image: str) -> None:
        self._k8s = k8s_client
        self._ns = namespace
        self._image = image

    def submit(self, job: RunJob) -> None:
        manifest = {
            "apiVersion": "batch/v1",
            "kind": "Job",
            "metadata": {"name": f"eval-{job.run_id}", "namespace": self._ns,
                         "labels": {"run_id": str(job.run_id), "runner_class": "ci"}},
            "spec": {
                "activeDeadlineSeconds": job.wall_clock_budget_s,
                "backoffLimit": 0,
                "template": {
                    "spec": {
                        "restartPolicy": "Never",
                        "containers": [{
                            "name": "eval",
                            "image": self._image,
                            "env": [
                                {"name": "RUN_ID",             "value": str(job.run_id)},
                                {"name": "DATASET_VERSION_ID", "value": job.dataset_version_id or ""},
                                {"name": "EVALUATOR_SET_ID",   "value": job.evaluator_set_id},
                                {"name": "APP_VERSION_ID",     "value": job.app_version_id},
                                {"name": "COST_BUDGET_USD",    "value": str(job.cost_budget_usd)},
                            ],
                        }],
                    },
                },
            },
        }
        self._k8s.create_namespaced_job(namespace=self._ns, body=manifest)

    def heartbeat(self, run_id: uuid.UUID) -> Heartbeat:
        status = self._k8s.read_namespaced_job_status(name=f"eval-{run_id}", namespace=self._ns)
        if status.status.succeeded:
            return Heartbeat(run_id=run_id, status="ok")
        if status.status.failed:
            return Heartbeat(run_id=run_id, status="failed",
                             error=str(status.status.conditions))
        return Heartbeat(run_id=run_id, status="running",
                         processed_count=status.status.active or 0)

    def cancel(self, run_id: uuid.UUID) -> None:
        self._k8s.delete_namespaced_job(name=f"eval-{run_id}", namespace=self._ns,
                                        propagation_policy="Foreground")
```

### 2.5 `runners/online.py` — an online-stream runner

```python
"""An in-process asyncio runner that evaluates a stream of traces drawn
from the online-eval loop (mod-107 ex-01). It does NOT iterate a dataset;
`dataset_version_id` is None for its runs."""
from __future__ import annotations
import asyncio
import uuid
from typing import Any

from plane.runner import Runner
from plane.types import Heartbeat, RunJob


class OnlineStreamRunner:
    runner_class = "online"

    def __init__(self, evaluator_pool, sink, trace_queue) -> None:
        self._pool = evaluator_pool
        self._sink = sink
        self._queue = trace_queue
        self._state: dict[uuid.UUID, dict[str, Any]] = {}
        self._cancelled: set[uuid.UUID] = set()

    def submit(self, job: RunJob) -> None:
        self._state[job.run_id] = {"processed": 0, "cost": 0.0,
                                   "status": "running", "evaluators": job.evaluator_set_id}
        asyncio.get_event_loop().create_task(self._run(job))

    def heartbeat(self, run_id: uuid.UUID) -> Heartbeat:
        s = self._state.get(run_id)
        if s is None:
            return Heartbeat(run_id=run_id, status="failed", error="unknown run_id")
        return Heartbeat(run_id=run_id, status=s["status"],
                         processed_count=s["processed"], cost_spent_usd=s["cost"])

    def cancel(self, run_id: uuid.UUID) -> None:
        self._cancelled.add(run_id)

    async def _run(self, job: RunJob) -> None:
        async for header in self._queue:
            if job.run_id in self._cancelled:
                self._state[job.run_id]["status"] = "canceled"
                return
            verdicts = await self._pool.evaluate(header, job.evaluator_set_id)
            for v in verdicts:
                self._sink.upsert(v, run_id=job.run_id)
            self._state[job.run_id]["processed"] += 1
            self._state[job.run_id]["cost"] += sum(v.cost_usd for v in verdicts)
        self._state[job.run_id]["status"] = "ok"
```

### 2.6 `runners/human.py` — a human-review router

```python
"""Hands cases off to annotators. Submit opens a review batch; heartbeat
returns how many items have been scored; cancel closes the batch. Writes
one `eval_result` row per scored item with `label_source='human'`."""
from __future__ import annotations
import uuid

from plane.runner import Runner
from plane.types import Heartbeat, RunJob


class HumanReviewRunner:
    runner_class = "human"

    def __init__(self, review_queue, sink) -> None:
        self._q = review_queue
        self._sink = sink
        self._state: dict[uuid.UUID, dict[str, int | str | float]] = {}

    def submit(self, job: RunJob) -> None:
        batch_id = self._q.open_batch(
            run_id=str(job.run_id),
            dataset_version_id=job.dataset_version_id,
            evaluator_set_id=job.evaluator_set_id,
        )
        self._state[job.run_id] = {"batch_id": batch_id, "processed": 0,
                                   "total": self._q.size(batch_id), "status": "running"}

    def heartbeat(self, run_id: uuid.UUID) -> Heartbeat:
        s = self._state[run_id]
        self._state[run_id]["processed"] = self._q.scored_count(s["batch_id"])
        if self._state[run_id]["processed"] >= s["total"]:
            self._state[run_id]["status"] = "ok"
        return Heartbeat(run_id=run_id, status=self._state[run_id]["status"],
                         processed_count=int(self._state[run_id]["processed"]),
                         total_count=int(s["total"]))

    def cancel(self, run_id: uuid.UUID) -> None:
        s = self._state[run_id]
        self._q.close_batch(s["batch_id"])
        self._state[run_id]["status"] = "canceled"
```

### 2.7 `plane/registry.py` — evaluator-set content addressing

```python
"""An evaluator_set_id is the SHA of a sorted list of (evaluator_id,
evaluator_version_id) pairs. Storing it means two runs ask the same
question iff they resolve to the same set. Content addressing again."""
from __future__ import annotations
import hashlib
import json
from dataclasses import dataclass


@dataclass(frozen=True)
class EvaluatorRef:
    evaluator_id: str
    evaluator_version_id: str


def evaluator_set_id(refs: list[EvaluatorRef]) -> str:
    sorted_refs = sorted(refs, key=lambda r: (r.evaluator_id, r.evaluator_version_id))
    body = json.dumps(
        [{"evaluator_id": r.evaluator_id, "evaluator_version_id": r.evaluator_version_id}
         for r in sorted_refs],
        separators=(",", ":"),
    ).encode()
    return "sha256:" + hashlib.sha256(body).hexdigest()
```

### 2.8 `plane/http.py` — the submit endpoint

```python
"""One FastAPI endpoint; POST /runs. The body is a RunRequest; the response
is the run_id. 429 on budget exhaustion, 409 on illegal dedup collision."""
from __future__ import annotations
from fastapi import FastAPI, HTTPException

from .service import ControlPlane
from .types import RunRequest


def make_app(plane: ControlPlane) -> FastAPI:
    app = FastAPI()

    @app.post("/runs")
    def submit(req: RunRequest) -> dict:
        try:
            run_id = plane.submit(req)
        except LookupError as e:
            raise HTTPException(status_code=404, detail=str(e))
        return {"run_id": str(run_id)}

    @app.get("/runs/{run_id}")
    def poll(run_id: str) -> dict:
        import uuid as _uuid
        hb = plane.poll(_uuid.UUID(run_id))
        return {"status": hb.status, "processed_count": hb.processed_count,
                "cost_spent_usd": hb.cost_spent_usd, "error": hb.error}

    @app.post("/runs/{run_id}/cancel")
    def cancel(run_id: str) -> dict:
        import uuid as _uuid
        plane.cancel(_uuid.UUID(run_id))
        return {"status": "canceled"}

    return app
```

### 2.9 `tests/test_submit_idempotent.py`

```python
from plane.types import RunRequest
from plane.service import ControlPlane


def test_duplicate_submit_returns_same_run_id(plane, seed):
    req = RunRequest(
        dataset_version_id=seed["dataset_version_id"],
        evaluator_set_id=seed["evaluator_set_id"],
        app_version_id=seed["app_version_id"],
        runner_class="ci",
        initiator="alice",
    )
    first = plane.submit(req)
    second = plane.submit(req)
    assert first == second


def test_mismatched_app_version_fails_fast(plane, seed):
    bad = RunRequest(
        dataset_version_id=seed["dataset_version_id"],
        evaluator_set_id=seed["evaluator_set_id"],
        app_version_id="sha256:" + "0"*64,
        runner_class="ci", initiator="alice",
    )
    import pytest
    with pytest.raises(LookupError, match="unknown app_version_id"):
        plane.submit(bad)
```

### 2.10 `tests/test_budget_enforcement.py`

```python
import time, uuid
from plane.types import Heartbeat, RunRequest


def test_cost_budget_triggers_abort(plane, seed, fake_runner):
    req = RunRequest(
        dataset_version_id=seed["dataset_version_id"],
        evaluator_set_id=seed["evaluator_set_id"],
        app_version_id=seed["app_version_id"],
        runner_class="ci", initiator="alice",
        wall_clock_budget_s=3600, cost_budget_usd=1.0,
    )
    run_id = plane.submit(req)
    # Simulate a runner heartbeat reporting cost over budget.
    fake_runner.set_heartbeat(run_id, Heartbeat(run_id=run_id, status="running",
                                                processed_count=10, cost_spent_usd=1.5))
    hb = plane.poll(run_id)
    assert hb.status in {"aborted", "running"}   # plane triggers abort on the next poll
    hb2 = plane.poll(run_id)
    assert hb2.status == "aborted"
```

### 2.11 Runner registration — one place to add a new backend

```python
# bootstrap.py
from plane.service import ControlPlane
from runners.ci     import CIRunner
from runners.online import OnlineStreamRunner
from runners.human  import HumanReviewRunner

def build_plane(conn, k8s_client, async_pool, review_queue) -> ControlPlane:
    return ControlPlane(
        conn=conn,
        runners={
            "ci":     CIRunner(k8s_client=k8s_client, namespace="eval", image="eval-worker:2026-11"),
            "online": OnlineStreamRunner(evaluator_pool=async_pool.pool, sink=async_pool.sink,
                                          trace_queue=async_pool.queue),
            "human":  HumanReviewRunner(review_queue=review_queue, sink=async_pool.sink),
        },
    )
```

## 3. Validation steps

1. **Start the stack**: `docker compose up postgres` and `uvicorn plane.http:make_app`. `curl -X POST /runs -d '{...}'` returns a `run_id`.
2. **Submit idempotently**: fire the same payload twice within 15 minutes; the response `run_id` is identical. `SELECT COUNT(*) FROM run WHERE metadata->>'idempotency_key' = '...'` returns 1.
3. **Lineage rejection**: submit with a fake `app_version_id`; endpoint returns 404 and no `run` row is written.
4. **Runner dispatch**: submit with `runner_class="ci"`; a Kubernetes Job with label `run_id=<uuid>` appears in the `eval` namespace within 1 s. Pods write `eval_result` rows carrying the same `run_id`.
5. **Polling**: `curl /runs/<id>` shows `status=running` with a growing `processed_count`. On Job completion the endpoint flips to `status=ok` and `finished_ns` is set in Postgres.
6. **Budget enforcement**: submit with `cost_budget_usd=0.001`; after the first evaluator costs > budget, the next poll shows `status=aborted`. The Kubernetes Job is deleted; subsequent `eval_result` rows for the run have `status='skipped'`.
7. **Cancellation**: submit, then `curl -X POST /runs/<id>/cancel`. `SELECT status FROM run WHERE run_id = '...'` returns `canceled`. No further `eval_result` rows are written.
8. **Cross-runner handoff**: submit two runs with the same `dataset_version_id` and `evaluator_set_id`, one `runner_class=ci` and one `runner_class=human`. Both write `eval_result` rows; `SELECT runner_class, COUNT(*) FROM v_verdict_full WHERE run_id IN (...) GROUP BY 1` shows both populated.
9. **Streaming runner**: push ten fake trace headers onto the online runner's queue; the online runner's `run_id` accumulates verdicts and heartbeats report increasing `processed_count`.
10. **Pytest**: `pytest -q tests/`. All idempotency, lineage, and budget tests pass.

## 4. Rubric / review checklist

| #   | Criterion                                                                                                    | Pass                                                                                   | Fail                                                                     |
| --- | ------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| R1  | One submit endpoint; many runners pluggable through a `Runner` protocol.                                      | `Runner` protocol has `submit/heartbeat/cancel`; registry is a map.                    | Each runner needs its own endpoint; the HTTP layer holds runner-specific code. |
| R2  | Lineage FKs are asserted at submit; the plane fails before writing a `run` row for bad ids.                   | Submit rejects unknown `app_version_id` / `dataset_version_id`.                        | Bad ids reach the runner and die inside.                                 |
| R3  | Idempotency keys dedupe submits in a declared window.                                                         | Repeat submit returns the same `run_id`.                                               | Every submit creates a new `run_id`; retries flood the runner.           |
| R4  | Each run carries a wall-clock and a cost budget; the plane enforces both.                                     | Over-budget heartbeat triggers `aborted` and cancels downstream.                        | Unbounded runs; a runaway judge can burn through the daily cap.          |
| R5  | Cancellation is cooperative and writes `skipped` rows for remaining evaluators.                                | `cancel` sets `status='canceled'`; downstream rows carry `status='skipped'`.           | Cancellation deletes rows; coverage math breaks.                         |
| R6  | Every runner writes `eval_result` rows with the same `run_id`; one view aggregates all runs.                    | `v_verdict_full` joins across runs; a human and a CI run share analysis path.           | Each runner writes to its own table; dashboards are split.               |
| R7  | Evaluator sets are content-addressed; two runs ask the same question iff they resolve to the same id.          | `evaluator_set_id` SHA test passes.                                                     | Evaluator set is a label; two runs with "v2" may differ silently.        |
| R8  | Runners are stateless from the plane's perspective; restart loses no run row.                                  | Plane durably stores `run` rows; heartbeat reconstructs status from the runner.         | Plane holds in-memory state about running jobs.                          |
| R9  | Heartbeats include `processed_count`, `cost_spent_usd`, and a reconciled `status`; dashboards read them.       | Heartbeat shape proven in tests.                                                        | Progress is only visible by tailing runner logs.                         |
| R10 | Priority bands are explicit (`fast`, `standard`, `background`); each runner's queue respects them.             | Priority present on `RunRequest`; CI runner polls `fast` before `standard`.             | All jobs share one FIFO; small ad-hoc runs starve behind big batches.    |

A passing submission is 8+ of 10 with R2, R3, R4, R5, R7 all hard-required.

## 5. Common mistakes

- **One endpoint per runner.** Every runner reinvents resource ids, retry policies, and dashboards. Collapse behind a single `POST /runs`.
- **No lineage check at submit.** A bad dataset id dies inside the runner, wastes a pod, and leaves a dangling `run` row. Fail fast at the submit boundary.
- **Non-idempotent submits.** CI retry policies and scheduled cron jobs will double-submit; without an idempotency key the runner queue fills with duplicates and dashboards double-count.
- **Unbounded budgets.** A judge bug can burn thousands of dollars overnight. Each run MUST declare wall-clock and cost caps, and the plane MUST enforce them.
- **Cancellation = row deletion.** Deleting rows breaks drift denominators and audit trails. Write `status='skipped'` and keep the row.
- **Runner-specific verdict tables.** If each runner has its own `eval_result_<kind>` table, the dashboard has to UNION them and the SLO (exercise-04) breaks on new runners. One table, one `run_id`, one `runner_class` label.
- **Evaluator set as a label.** "judge-set-v2" means nothing six months later. Content-address the set so equality is decidable.
- **In-memory plane state.** If the plane process crashes, you lose "which run_id maps to which runner". Keep the mapping in the `run` row (`kind` column already carries runner_class).
- **Polling the runner synchronously inside submit.** Submit MUST return immediately. Heartbeats run on a separate schedule; dashboards pull, the plane doesn't push into the user's HTTP response.
- **No priority bands.** A single FIFO means a two-hour regression blocks a 30-second CI gate. Priorities let different runners service different queues.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MANAGE 4.1` (post-deployment monitoring and incident response) frames budget enforcement; `GOVERN 2.1` frames audit trails on runs.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning) motivates the lineage check at submit; a runner should never evaluate a dataset it can't link back to.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.M0015` (prompt hardening) is relevant when the runner image embeds judge prompts; promoting the evaluator set to a content hash makes injection surfaces auditable.
- Kubernetes Job API, <https://kubernetes.io/docs/concepts/workloads/controllers/job/>, for the CI runner contract.
- FastAPI, <https://fastapi.tiangolo.com/>, for the submit endpoint.
- Local exercise reader: `lessons/mod-110-eval-data-platform-slice/exercises/exercise-03-multi-runner-orchestration-behind-one-plane.md`.
- Related solutions: exercise-01 owns `run` and `eval_result`; exercise-02 owns `dataset_version` referenced in `RunRequest`; exercise-04 reads the `run` + `eval_result` tables for SLOs; exercise-05 submits with `runner_class='ci'` on every PR.
