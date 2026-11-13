# exercise-01 — Sampled-Trace Online-Eval Loop — Solution

> Reference solution for `lessons/mod-107-online-eval-and-regression/exercises/exercise-01-sampled-trace-online-eval-loop.md`.
> The exercise turns live app traces into a stream of eval verdicts without rerunning the model — a sampler picks which traces to score, an evaluator worker pool runs judges/rules asynchronously, and a sink writes verdicts back where dashboards (exercise-06), drift alerts (exercise-02), and canary gates (exercise-03) read them.

## 1. Solution overview

The deliverable is a Python package `online_eval/` with four seams:

1. **Sampler** — reads trace headers from a queue; decides which traces to score. Strategy is stratified + reservoir, biased toward high-risk segments and low-sample tool shapes. It never reads the full trace payload at this stage; sampling happens on metadata so the decision is cheap and streamable.
2. **Fetcher** — given a sampled `trace_id`, pulls the trace payload from the tracing backend (OpenTelemetry collector, Langfuse, Phoenix). Hides PII fields per the policy before handing the trace to the evaluator.
3. **Evaluator pool** — a thread-pool (or asyncio task group) that runs a configured set of evaluators on each trace. Evaluators are pure functions `(trace, policy) -> list[Verdict]`. Each verdict carries evaluator id + version, score, confidence, latency_ms, and cost_usd.
4. **Sink** — writes verdicts to the eval store, tagged with the same `trace_id` so the UI can link verdict back to trace. Emits per-evaluator counters (processed, failed, over-budget) for the dashboards.

Five design choices anchor the solution:

1. **Sample on metadata, fetch on demand.** The sampler never sees the full trace. This keeps the fan-out point cheap, keeps the PII surface small, and lets you throttle fetches independently from sampling rate.
2. **Stratified sampler, not uniform.** A uniform sampler at 1% will miss rare-but-important tool shapes forever. The stratum is `(tenant_class, surface, tool_shape, judge_cost_bucket)`; each stratum has a floor rate so low-volume high-risk slices are guaranteed representation. This is the `measure` function in [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework) — coverage must be deliberate.
3. **Judge calls are rate-limited and budgeted per-stratum.** A runaway judge bill is a reliability incident, not a line item. Each evaluator declares a monthly token budget; the loop exports remaining-budget as a counter and switches that evaluator to sample-only-cheap mode when below 10%.
4. **Idempotent by `(trace_id, evaluator_id, evaluator_version)`.** Replays, retries, and backfills must not double-count. The sink's primary key is this triple; a reprocess updates the row rather than inserting a duplicate.
5. **At-least-once with deterministic dedup.** Delivery semantics are at-least-once (you cannot avoid this when the fetch and the judge are independent network hops). The sink's primary key provides the dedup. This is also what lets exercise-04's confidence sequences stay valid under replay.

## 2. Worked implementation

```
mod-107/exercise-01/
├── pyproject.toml
├── README.md
├── online_eval/
│   ├── __init__.py
│   ├── config.py            # typed config, loaded from YAML
│   ├── sampler.py           # stratified + reservoir sampler
│   ├── fetcher.py           # trace fetch + PII redaction
│   ├── evaluators/
│   │   ├── __init__.py
│   │   ├── base.py          # Evaluator protocol + Verdict dataclass
│   │   ├── rule_schema.py   # cheap structural checks
│   │   ├── judge.py         # llm-as-judge wrapper
│   │   └── registry.py
│   ├── pool.py              # evaluator worker pool
│   ├── sink.py              # Verdict sink (local sqlite for the exercise)
│   ├── metrics.py           # OpenTelemetry metric registration
│   └── cli.py               # `python -m online_eval.cli`
├── policy/
│   ├── pii_fields.yaml
│   └── strata.yaml
├── fixtures/
│   ├── traces.jsonl         # 1k synthetic trace headers + payloads
│   └── expected_verdicts.json
└── tests/
    ├── test_sampler.py
    ├── test_fetcher_redaction.py
    ├── test_pool.py
    └── test_sink_idempotence.py
```

### 2.1 `online_eval/config.py` — typed config

```python
"""Config dataclasses. Loaded from policy/*.yaml; no Pydantic required."""
from __future__ import annotations
from dataclasses import dataclass, field


@dataclass(frozen=True)
class StratumRule:
    key: tuple[str, ...]           # (tenant_class, surface, tool_shape, cost_bucket)
    floor_rate: float              # min sampling rate (e.g. 0.10 for 10%)
    ceiling_per_hour: int          # hard cap on sampled traces per hour
    evaluators: tuple[str, ...]    # evaluator ids to run on this stratum


@dataclass(frozen=True)
class EvaluatorBudget:
    evaluator_id: str
    monthly_token_budget: int
    cheap_mode_threshold: float = 0.10  # switch to sample-only at 10% remaining


@dataclass(frozen=True)
class SamplerConfig:
    default_rate: float
    strata: tuple[StratumRule, ...]
    reservoir_size: int                   # per-stratum reservoir for backfills
    base_salt: str                        # deterministic hashing seed


@dataclass(frozen=True)
class PIIPolicy:
    redact_fields: frozenset[str]
    hash_fields: frozenset[str]           # hash-and-keep (user_id, session_id)
    drop_fields: frozenset[str]           # drop entirely (ssn, address)


@dataclass(frozen=True)
class OnlineEvalConfig:
    sampler: SamplerConfig
    budgets: tuple[EvaluatorBudget, ...]
    pii: PIIPolicy
    sink_dsn: str
    worker_count: int = 8
    fetch_timeout_s: float = 3.0
```

### 2.2 `online_eval/sampler.py` — stratified + reservoir

```python
"""Stratified + reservoir sampler.

Decision inputs are trace HEADERS (metadata only), not payloads. A header
carries: trace_id, tenant_id, tenant_class, surface, tool_shape, cost_bucket,
wall_ns. The sampler's job is to return `(sample: bool, evaluator_ids: list[str])`
without touching tokens, user messages, or tool arguments.
"""
from __future__ import annotations
import hashlib
import time
from collections import defaultdict, deque
from dataclasses import dataclass
from typing import Any

from .config import SamplerConfig, StratumRule


@dataclass
class _Window:
    """Fixed-capacity sliding window of sample timestamps for rate limiting."""
    capacity: int
    stamps: deque[float]

    def allow(self, now_s: float, window_s: float = 3600.0) -> bool:
        while self.stamps and now_s - self.stamps[0] > window_s:
            self.stamps.popleft()
        if len(self.stamps) < self.capacity:
            self.stamps.append(now_s)
            return True
        return False


class StratifiedSampler:
    def __init__(self, cfg: SamplerConfig) -> None:
        self._cfg = cfg
        self._windows: dict[tuple[str, ...], _Window] = {}
        self._reservoir: dict[tuple[str, ...], deque[str]] = defaultdict(
            lambda: deque(maxlen=cfg.reservoir_size)
        )

    def _stratum_key(self, header: dict[str, Any]) -> tuple[str, ...]:
        return (
            header["tenant_class"], header["surface"],
            header["tool_shape"], header["cost_bucket"],
        )

    def _rule_for(self, key: tuple[str, ...]) -> StratumRule | None:
        for rule in self._cfg.strata:
            if rule.key == key:
                return rule
        return None

    def _coin(self, header: dict[str, Any], rate: float) -> bool:
        """Deterministic hash-based coin. Same trace_id → same decision across restarts."""
        h = hashlib.sha256(
            (self._cfg.base_salt + header["trace_id"]).encode()
        ).digest()
        bucket = int.from_bytes(h[:8], "big") / 2**64
        return bucket < rate

    def decide(self, header: dict[str, Any], now_s: float | None = None) -> tuple[bool, list[str]]:
        if now_s is None:
            now_s = time.monotonic()
        key = self._stratum_key(header)
        rule = self._rule_for(key)
        rate = rule.floor_rate if rule else self._cfg.default_rate
        evaluators = list(rule.evaluators) if rule else []
        if not self._coin(header, rate):
            # Keep a reservoir sample for low-volume strata: backfill job
            # later picks from the reservoir if the stratum saw < N traces this hour.
            self._reservoir[key].append(header["trace_id"])
            return False, []
        # Rate-limit
        window = self._windows.setdefault(
            key, _Window(capacity=rule.ceiling_per_hour if rule else 10_000,
                         stamps=deque())
        )
        if not window.allow(now_s):
            return False, []
        return True, evaluators

    def drain_reservoir(self, key: tuple[str, ...], min_samples: int) -> list[str]:
        """Called by a 5-minute backfill job if a stratum's hourly count is under floor.
        Yields trace_ids that were rejected by the coin but held in the reservoir."""
        held = list(self._reservoir[key])
        self._reservoir[key].clear()
        return held[:min_samples]
```

### 2.3 `online_eval/fetcher.py` — fetch + PII redaction

```python
"""Trace fetcher. In the exercise fixture the backend is a JSONL file; in
production it's the tracing backend's query API (Phoenix, Langfuse, Jaeger).
The seam is `fetch(trace_id) -> dict`; swap the impl for production."""
from __future__ import annotations
import hashlib
import json
from pathlib import Path
from typing import Any

from .config import PIIPolicy


class TraceFetcher:
    def __init__(self, source: Path, pii: PIIPolicy) -> None:
        self._source = source
        self._pii = pii
        # Build a small in-memory index for the exercise fixture.
        self._index: dict[str, dict[str, Any]] = {}
        with source.open() as f:
            for line in f:
                row = json.loads(line)
                self._index[row["trace_id"]] = row

    def fetch(self, trace_id: str) -> dict[str, Any] | None:
        row = self._index.get(trace_id)
        if row is None:
            return None
        return self._redact(row)

    def _redact(self, trace: dict[str, Any]) -> dict[str, Any]:
        """Walk the trace tree; apply drop / hash / keep rules per field name."""
        def _walk(node: Any) -> Any:
            if isinstance(node, dict):
                out: dict[str, Any] = {}
                for k, v in node.items():
                    if k in self._pii.drop_fields:
                        continue
                    if k in self._pii.hash_fields and isinstance(v, str):
                        out[k] = "h:" + hashlib.sha256(v.encode()).hexdigest()[:16]
                        continue
                    if k in self._pii.redact_fields:
                        out[k] = "[REDACTED]"
                        continue
                    out[k] = _walk(v)
                return out
            if isinstance(node, list):
                return [_walk(x) for x in node]
            return node
        return _walk(trace)
```

### 2.4 `online_eval/evaluators/base.py` — Evaluator protocol

```python
"""Evaluator protocol and Verdict dataclass. Downstream readers (sink,
drift job, canary gate) rely on this shape."""
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Any, Protocol


@dataclass(frozen=True)
class Verdict:
    trace_id: str
    evaluator_id: str
    evaluator_version: str
    score: float | None             # judge score, bool cast to 0/1, or None on error
    label: str | None               # optional categorical outcome
    confidence: float | None        # [0, 1] when evaluator is a probabilistic judge
    reason: str | None              # short free-text explanation (judge rationale)
    latency_ms: int
    cost_usd: float
    status: str                     # "ok" | "timeout" | "error" | "over_budget"
    attributes: dict[str, Any] = field(default_factory=dict)


class Evaluator(Protocol):
    evaluator_id: str
    evaluator_version: str

    def evaluate(self, trace: dict[str, Any]) -> Verdict: ...
```

### 2.5 `online_eval/evaluators/rule_schema.py` — cheap structural check

```python
"""An evaluator that checks a trace against schema rules — tool-argument
types, forbidden tool calls, empty assistant replies, etc. Costs zero USD
and runs synchronously; this is the baseline every trace gets."""
from __future__ import annotations
import time
from typing import Any

from .base import Verdict


class SchemaRuleEvaluator:
    evaluator_id = "rule.schema"
    evaluator_version = "v1.2"

    def __init__(self, forbidden_tools: frozenset[str]) -> None:
        self._forbidden = forbidden_tools

    def evaluate(self, trace: dict[str, Any]) -> Verdict:
        start = time.monotonic_ns()
        violations: list[str] = []
        for span in trace.get("spans", []):
            if span.get("kind") != "TOOL":
                continue
            if span.get("name", "") in self._forbidden:
                violations.append(f"forbidden_tool:{span['name']}")
            if span.get("arguments") is None and span.get("status") == "ok":
                violations.append(f"missing_arguments:{span['name']}")
        latency_ms = (time.monotonic_ns() - start) // 1_000_000
        passed = not violations
        return Verdict(
            trace_id=trace["trace_id"],
            evaluator_id=self.evaluator_id,
            evaluator_version=self.evaluator_version,
            score=1.0 if passed else 0.0,
            label="pass" if passed else "fail",
            confidence=1.0,
            reason=None if passed else ";".join(violations),
            latency_ms=latency_ms,
            cost_usd=0.0,
            status="ok",
            attributes={"violation_count": len(violations)},
        )
```

### 2.6 `online_eval/evaluators/judge.py` — llm-as-judge wrapper

```python
"""Thin wrapper over an LLM-judge. The judge itself is injected so tests
can substitute a deterministic fake. Budget enforcement and timeout are
centralised here."""
from __future__ import annotations
import time
from typing import Any, Callable

from .base import Verdict


JudgeFn = Callable[[dict[str, Any]], tuple[float, str, float, float]]
# Returns (score, reason, confidence, cost_usd)


class JudgeEvaluator:
    def __init__(
        self,
        evaluator_id: str,
        evaluator_version: str,
        judge_fn: JudgeFn,
        timeout_s: float,
        budget_remaining_cb: Callable[[], float],
    ) -> None:
        self.evaluator_id = evaluator_id
        self.evaluator_version = evaluator_version
        self._fn = judge_fn
        self._timeout_s = timeout_s
        self._budget_remaining = budget_remaining_cb

    def evaluate(self, trace: dict[str, Any]) -> Verdict:
        start = time.monotonic_ns()
        if self._budget_remaining() <= 0:
            return Verdict(
                trace_id=trace["trace_id"],
                evaluator_id=self.evaluator_id,
                evaluator_version=self.evaluator_version,
                score=None, label=None, confidence=None, reason="budget_exhausted",
                latency_ms=0, cost_usd=0.0, status="over_budget",
            )
        try:
            score, reason, confidence, cost = self._fn(trace)
            latency_ms = (time.monotonic_ns() - start) // 1_000_000
            return Verdict(
                trace_id=trace["trace_id"],
                evaluator_id=self.evaluator_id,
                evaluator_version=self.evaluator_version,
                score=score, label=None, confidence=confidence, reason=reason,
                latency_ms=latency_ms, cost_usd=cost, status="ok",
            )
        except TimeoutError:
            return Verdict(
                trace_id=trace["trace_id"],
                evaluator_id=self.evaluator_id,
                evaluator_version=self.evaluator_version,
                score=None, label=None, confidence=None, reason="timeout",
                latency_ms=int(self._timeout_s * 1000), cost_usd=0.0, status="timeout",
            )
        except Exception as exc:  # noqa: BLE001
            return Verdict(
                trace_id=trace["trace_id"],
                evaluator_id=self.evaluator_id,
                evaluator_version=self.evaluator_version,
                score=None, label=None, confidence=None, reason=f"error:{type(exc).__name__}",
                latency_ms=(time.monotonic_ns() - start) // 1_000_000,
                cost_usd=0.0, status="error",
            )
```

### 2.7 `online_eval/pool.py` — worker pool

```python
"""Evaluator pool. One concurrent queue of (trace_id, evaluator_ids) pairs;
N workers pull, fetch, run each evaluator, and write verdicts to the sink.

For the exercise a ThreadPoolExecutor is fine (judges are I/O-bound). In
production the same seam accepts an asyncio TaskGroup or a Celery worker."""
from __future__ import annotations
import logging
from concurrent.futures import ThreadPoolExecutor
from typing import Any

from .evaluators.base import Evaluator, Verdict
from .fetcher import TraceFetcher
from .sink import VerdictSink

log = logging.getLogger(__name__)


class EvaluatorPool:
    def __init__(
        self,
        fetcher: TraceFetcher,
        sink: VerdictSink,
        evaluators: dict[str, Evaluator],
        worker_count: int,
    ) -> None:
        self._fetcher = fetcher
        self._sink = sink
        self._evaluators = evaluators
        self._executor = ThreadPoolExecutor(max_workers=worker_count)

    def submit(self, trace_id: str, evaluator_ids: list[str]) -> None:
        self._executor.submit(self._run_one, trace_id, evaluator_ids)

    def _run_one(self, trace_id: str, evaluator_ids: list[str]) -> None:
        trace = self._fetcher.fetch(trace_id)
        if trace is None:
            log.warning("trace_missing", extra={"trace_id": trace_id})
            return
        for ev_id in evaluator_ids:
            ev = self._evaluators.get(ev_id)
            if ev is None:
                log.warning("evaluator_missing", extra={"evaluator_id": ev_id})
                continue
            verdict = ev.evaluate(trace)
            self._sink.upsert(verdict)

    def shutdown(self, wait: bool = True) -> None:
        self._executor.shutdown(wait=wait)
```

### 2.8 `online_eval/sink.py` — idempotent sink

```python
"""Verdict sink. Primary key is (trace_id, evaluator_id, evaluator_version);
upsert so retries and replays do not double-count. SQLite is used for the
exercise; the same DDL maps cleanly onto Postgres/ClickHouse."""
from __future__ import annotations
import json
import sqlite3
from dataclasses import asdict

from .evaluators.base import Verdict


DDL = """
CREATE TABLE IF NOT EXISTS verdict (
    trace_id          TEXT NOT NULL,
    evaluator_id      TEXT NOT NULL,
    evaluator_version TEXT NOT NULL,
    score             REAL,
    label             TEXT,
    confidence        REAL,
    reason            TEXT,
    latency_ms        INTEGER NOT NULL,
    cost_usd          REAL NOT NULL,
    status            TEXT NOT NULL,
    attributes        TEXT NOT NULL,          -- JSON
    first_seen_ns     INTEGER NOT NULL,
    last_updated_ns   INTEGER NOT NULL,
    PRIMARY KEY (trace_id, evaluator_id, evaluator_version)
);

CREATE INDEX IF NOT EXISTS idx_verdict_by_evaluator
    ON verdict (evaluator_id, evaluator_version, last_updated_ns);
"""


class VerdictSink:
    def __init__(self, dsn: str) -> None:
        self._conn = sqlite3.connect(dsn, check_same_thread=False)
        self._conn.executescript(DDL)

    def upsert(self, v: Verdict, *, now_ns: int) -> None:
        payload = asdict(v)
        self._conn.execute(
            """
            INSERT INTO verdict (
                trace_id, evaluator_id, evaluator_version, score, label, confidence,
                reason, latency_ms, cost_usd, status, attributes,
                first_seen_ns, last_updated_ns
            ) VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
            ON CONFLICT (trace_id, evaluator_id, evaluator_version) DO UPDATE SET
                score = excluded.score,
                label = excluded.label,
                confidence = excluded.confidence,
                reason = excluded.reason,
                latency_ms = excluded.latency_ms,
                cost_usd = excluded.cost_usd,
                status = excluded.status,
                attributes = excluded.attributes,
                last_updated_ns = excluded.last_updated_ns
            """,
            (
                v.trace_id, v.evaluator_id, v.evaluator_version,
                v.score, v.label, v.confidence, v.reason,
                v.latency_ms, v.cost_usd, v.status,
                json.dumps(payload["attributes"], sort_keys=True),
                now_ns, now_ns,
            ),
        )
        self._conn.commit()
```

### 2.9 `tests/test_sampler.py` — determinism + coverage floors

```python
from online_eval.sampler import StratifiedSampler
from online_eval.config import SamplerConfig, StratumRule


def _cfg() -> SamplerConfig:
    return SamplerConfig(
        default_rate=0.01,
        strata=(
            StratumRule(
                key=("paid", "chat", "tool_call", "high"),
                floor_rate=1.0, ceiling_per_hour=1000,
                evaluators=("rule.schema", "judge.helpfulness"),
            ),
        ),
        reservoir_size=100, base_salt="exercise-01",
    )


def test_high_risk_stratum_hits_floor_rate():
    s = StratifiedSampler(_cfg())
    sampled = 0
    for i in range(1000):
        ok, evs = s.decide({
            "trace_id": f"t{i}", "tenant_class": "paid",
            "surface": "chat", "tool_shape": "tool_call", "cost_bucket": "high",
        }, now_s=0.0)
        if ok:
            sampled += 1
            assert "rule.schema" in evs
    assert sampled == 1000  # floor_rate=1.0


def test_default_stratum_samples_near_1_percent():
    s = StratifiedSampler(_cfg())
    sampled = 0
    for i in range(10_000):
        ok, _ = s.decide({
            "trace_id": f"t{i}", "tenant_class": "free",
            "surface": "chat", "tool_shape": "chat_only", "cost_bucket": "low",
        }, now_s=0.0)
        if ok:
            sampled += 1
    # 1% ± 0.3%
    assert 70 <= sampled <= 130


def test_decision_is_deterministic_across_restarts():
    cfg = _cfg()
    first = StratifiedSampler(cfg)
    second = StratifiedSampler(cfg)
    hdr = {"trace_id": "stable-id", "tenant_class": "free",
           "surface": "chat", "tool_shape": "chat_only", "cost_bucket": "low"}
    assert first.decide(hdr, now_s=0.0)[0] == second.decide(hdr, now_s=0.0)[0]
```

### 2.10 `tests/test_sink_idempotence.py`

```python
from online_eval.sink import VerdictSink
from online_eval.evaluators.base import Verdict


def test_upsert_is_idempotent(tmp_path):
    sink = VerdictSink(str(tmp_path / "v.db"))
    v = Verdict(
        trace_id="t1", evaluator_id="rule.schema", evaluator_version="v1.2",
        score=1.0, label="pass", confidence=1.0, reason=None,
        latency_ms=2, cost_usd=0.0, status="ok",
    )
    sink.upsert(v, now_ns=1)
    sink.upsert(v, now_ns=2)   # same PK, different timestamp → one row, updated timestamp
    rows = list(sink._conn.execute("SELECT COUNT(*), MAX(last_updated_ns) FROM verdict"))
    assert rows == [(1, 2)]
```

### 2.11 `policy/strata.yaml`

```yaml
# Floor rates read as: "at least this fraction of traces in this stratum are judged".
# High-risk strata (paid tenants, tool-calling traces, high cost) get full coverage.
default_rate: 0.01
reservoir_size: 500
base_salt: "prod-salt-rotate-on-pii-review"
strata:
  - key: [paid, chat, tool_call, high]
    floor_rate: 1.0
    ceiling_per_hour: 2000
    evaluators: [rule.schema, judge.helpfulness, judge.safety]
  - key: [paid, chat, tool_call, low]
    floor_rate: 0.10
    ceiling_per_hour: 1000
    evaluators: [rule.schema, judge.helpfulness]
  - key: [free, chat, chat_only, low]
    floor_rate: 0.01
    ceiling_per_hour: 500
    evaluators: [rule.schema]
```

## 3. Validation steps

1. **Install + run tests**: `pip install -e . && pytest -q tests/`. All four test modules pass; the sampler tests include a determinism check and a floor-rate coverage check.
2. **End-to-end smoke**: `python -m online_eval.cli --fixture fixtures/traces.jsonl --config policy/` reads the 1k synthetic headers, samples per strata, runs evaluators, and emits ~N verdicts to a local sqlite file.
3. **Determinism**: re-run step 2 with the same fixture and salt; the sqlite row count and verdict scores are byte-identical. This is required for exercise-04's confidence sequences to stay valid under replay.
4. **Coverage floor**: inspect `SELECT evaluator_id, COUNT(*) FROM verdict GROUP BY 1`. Every evaluator with a stratum rule has at least `floor_rate * stratum_volume` verdicts; no high-risk stratum is empty.
5. **Budget halting**: artificially set `monthly_token_budget=0` for one judge and rerun; the sink shows `status="over_budget"` rows for that evaluator and no cost_usd > 0 for it.
6. **PII redaction**: `jq '.spans[].inputs.ssn' fixtures/traces.jsonl | sort -u` is non-empty, but the same query on the sink's `attributes` is empty / `[REDACTED]`.
7. **Idempotence**: rerun the CLI a second time over the same fixture. Row count is unchanged; `last_updated_ns` advances.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Sampler decides on metadata only — no payload field is read until after `decide()` returns true. | `decide()` signature and tests use header-only dicts; no span bodies touched. | Sampler reads `span.content` or `prompt` before deciding. |
| R2 | Stratification is explicit and configurable; every stratum has a floor rate and a ceiling. | `policy/strata.yaml` present; each rule has both fields; defaults documented. | Flat uniform sample, or only ceilings without floors. |
| R3 | Sampling decision is deterministic given `(trace_id, base_salt)`. | `test_decision_is_deterministic_across_restarts` passes. | `random.random()` or `time.time()` in the sampler path. |
| R4 | PII redaction runs before the trace is handed to the evaluator pool. | Fetcher output never contains fields listed in `pii.redact_fields` / `drop_fields`. | Evaluator receives raw trace and redacts after judging. |
| R5 | Each evaluator declares a monthly budget; the loop stops it at zero and emits `status="over_budget"`. | `JudgeEvaluator` consults `budget_remaining_cb`; test proves zero-budget path. | Unbounded judge spend; silent failure when budget exhausted. |
| R6 | Verdict sink is idempotent under replay; PK is `(trace_id, evaluator_id, evaluator_version)`. | `test_upsert_is_idempotent` passes; reruns do not inflate counts. | Insert-only sink; reruns double verdict counts. |
| R7 | Evaluator version is part of the key, not a column. | `evaluator_version` appears in PK and in every Verdict. | Version only in `attributes`; a bump creates a silent overwrite. |
| R8 | Timeouts and errors are first-class statuses; they still produce a Verdict row. | `status` enum includes `timeout`, `error`, `over_budget`; rows written in each case. | Timeouts drop the row entirely, so the next drift job double-counts coverage. |
| R9 | No evaluator imports or calls a model-serving SDK at import time. | Grep for anthropic/openai imports returns only typed-protocol references in the judge wrapper. | Any eager model client in `__init__.py` or protocol module. |
| R10 | Metrics exported: sampled/sec, verdicts/sec by status, cost_usd by evaluator, remaining_budget by evaluator. | OpenTelemetry metric registration shown in `metrics.py`. | Counters absent; dashboard cannot render. |

A passing submission is 8+ of 10 with R1, R3, R4, R6, R8 all hard-required.

## 5. Common mistakes

- **Uniform sampling.** A flat 1% misses rare-but-dangerous slices forever. Stratify on `(tenant_class, surface, tool_shape, cost_bucket)` and declare floor rates for each high-risk stratum. See NIST AI RMF `MEASURE 2.3` on coverage.
- **Sampling on payload.** Reading the trace payload to decide whether to sample it defeats the point: you pay the fetch cost on 100% of traces. Keep the decision on metadata.
- **Non-deterministic sampling.** Using `random.random()` in the sampler makes backfills and replays diverge, which in turn invalidates exercise-04's confidence sequences. Hash on `(trace_id, salt)`.
- **No budget enforcement.** Judges are the second-largest line item in most AI apps; a runaway drift-job can spend four figures overnight. Declare budgets and halt when they hit zero.
- **Evaluator version as a column, not a key.** When you bump the judge prompt, the old verdicts should stay queryable. Including `evaluator_version` in the PK lets both coexist; putting it in a column silently overwrites history.
- **PII-redacting after the judge.** The judge call is the risky external hop. Redact in the fetcher, not after the sink. OWASP ML Top 10 `ML07: Transfer Learning Attack` and MITRE ATLAS's `AML.T0024` cover the exfiltration shape; both treat the first external call as the boundary.
- **At-most-once delivery.** Dropping a verdict on timeout corrupts drift and confidence-sequence math. Write a Verdict row with `status="timeout"` so the denominator is accurate.
- **Fetching inside the sampler.** Two separate concerns: deciding (cheap) and fetching (expensive). Combining them into one component means throttling one means throttling the other.
- **Dropping the reservoir.** Low-volume strata produce fewer than their floor rate suggests; a reservoir backfill at 5-minute intervals keeps the per-stratum coverage honest.
- **Logging full traces.** `log.info(trace)` undoes the redaction. Log the Verdict (which is already sanitised) and the `trace_id` only.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 2.3` (metric coverage) and `MANAGE 4.1` (post-deployment monitoring).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning) and `ML07` (transfer-learning attack) motivate the PII boundary in the fetcher.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0024: Exfiltration via LLM`; the sampled-trace loop is one of the controls listed under `AML.M0015: LLM Prompt Hardening`.
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/> (trace header fields used in `StratumRule.key`).
- Python standard library — `concurrent.futures.ThreadPoolExecutor`, <https://docs.python.org/3/library/concurrent.futures.html>.
- Local chapters and exercise reader: `lessons/mod-107-online-eval-and-regression/exercises/exercise-01-sampled-trace-online-eval-loop.md`.
- Downstream exercises in this module: exercise-02 (drift alerts) reads the sink; exercise-03 (canary) reads it per-cohort; exercise-04 (confidence sequences) needs the idempotence from R6; exercise-06 (dashboards) reads the metrics in R10.
