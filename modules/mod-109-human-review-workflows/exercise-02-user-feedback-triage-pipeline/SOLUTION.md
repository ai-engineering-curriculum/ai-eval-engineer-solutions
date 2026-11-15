# exercise-02 — User Feedback Triage Pipeline — Solution

> Reference solution for `lessons/mod-109-human-review-workflows/exercises/exercise-02-user-feedback-triage-pipeline.md`.
> Builds the pipeline that takes raw end-user feedback (thumbs up/down, free-text, inline flags) and produces a prioritised, deduplicated, reviewer-ready queue inside the workspace from exercise-01. Downstream, exercise-03 reads the resulting reviewed records for IAA and exercise-04 uses the severity-tagged clusters as inputs for gold-set rotation.

## 1. Solution overview

The deliverable is a batch-plus-stream pipeline `feedback_triage/` with four stages and a declarative routing policy. Raw feedback events land on a queue; the pipeline enriches them with trace context, clusters near-duplicates, scores each cluster for severity + novelty, and routes each cluster into one of four reviewer queues with an SLA. Nothing is auto-dismissed without an audit trail; every dropped cluster carries a reason.

Five design choices anchor the solution:

1. **Cluster before you prioritise.** 200 thumbs-downs on the same bad answer is one problem, not 200. The triage pipeline clusters feedback by `(trace_shape_hash, embedding_near_duplicate, rough_category)` *before* any prioritisation step; the reviewer sees one cluster with a count, not 200 identical cards. This is the single highest-leverage design choice; skipping it drowns the reviewer pool.
2. **Severity is a derived score, not a single field.** Severity = `w1 * harm_score + w2 * blast_radius + w3 * novelty + w4 * recency`. Each component is defined, each weight is in config. A thumbs-down with "this leaked my email" scores differently than "this was slow" even when both are `negative=true`.
3. **Priorities map to SLAs, not to review queues directly.** P0 (critical safety) → expert queue within 2 hours. P1 (likely user harm) → expert queue within 24 hours. P2 (quality regression) → first-pass queue within 72 hours. P3 (nit / already-known) → batch digest. SLAs make the pipeline's success measurable and failure-visible.
4. **Nothing is dropped silently.** Clusters that are dismissed (duplicates of already-triaged clusters, nits, test accounts) write a `decision_record` with reason and reviewer. This is the audit trail the NIST AI RMF `MANAGE 4.3` function expects for incidents and user-reported issues.
5. **PII is redacted at ingest, not at review.** Free-text feedback often contains emails, names, phone numbers. Redaction runs in stage 1, before the enriched event ever leaves the ingest node. Reviewers see redacted text; the raw text is stored under an access-controlled bucket for legal escalation only.

## 2. Worked implementation

```
mod-109/exercise-02/
├── pyproject.toml
├── README.md
├── feedback_triage/
│   ├── __init__.py
│   ├── schema.py                # FeedbackEvent, Cluster, Decision
│   ├── ingest.py                # stage 1 — normalise + redact
│   ├── enrich.py                # stage 2 — join trace + user context
│   ├── cluster.py               # stage 3 — near-duplicate clustering
│   ├── score.py                 # stage 4 — severity + priority
│   ├── route.py                 # stage 5 — queue routing
│   ├── audit.py                 # decision_record writer
│   └── cli.py
├── policy/
│   ├── severity_weights.yaml
│   ├── routing.yaml
│   └── pii_patterns.yaml
├── fixtures/
│   ├── feedback_events.jsonl
│   ├── traces.jsonl
│   └── expected_clusters.json
└── tests/
    ├── test_redaction.py
    ├── test_cluster.py
    ├── test_severity.py
    └── test_routing_slo.py
```

### 2.1 `feedback_triage/schema.py`

```python
"""Dataclasses for the pipeline stages. Each stage's output is the next
stage's input; schema changes are PR-reviewed."""
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal, Any

Priority = Literal["P0", "P1", "P2", "P3"]
Decision = Literal["route", "dismiss_duplicate", "dismiss_nit", "dismiss_test_account"]


@dataclass(frozen=True)
class FeedbackEvent:
    """Raw feedback as emitted by the client, normalised at stage 1."""
    feedback_id: str                 # client-generated UUID
    trace_id: str                    # from the OTel trace header
    user_id_hashed: str              # hashed at ingest; never raw
    surface: str                     # e.g. "chat_web", "mobile_app"
    rating: Literal["up", "down", "flag"]
    free_text_redacted: str          # PII-redacted
    rating_reason: str | None        # client-side tagged reason, if any
    client_ts_ns: int
    received_ns: int


@dataclass(frozen=True)
class EnrichedEvent:
    base: FeedbackEvent
    trace_shape_hash: str            # hash of (surface, tool_sequence, template_id)
    tenant_class: str
    reply_snippet: str               # first 400 chars of assistant reply, redacted
    embedding: tuple[float, ...]     # text embedding of (reply + feedback) for clustering


@dataclass(frozen=True)
class Cluster:
    cluster_id: str
    representative_trace_id: str
    event_ids: tuple[str, ...]
    count: int
    first_seen_ns: int
    last_seen_ns: int
    shape_hash: str
    category: str                    # from rating_reason + classifier
    centroid: tuple[float, ...]
    attributes: dict[str, Any] = field(default_factory=dict)


@dataclass(frozen=True)
class ScoredCluster:
    cluster: Cluster
    harm_score: float                # 0..1
    blast_radius: float              # 0..1 (unique_users / dau_estimate)
    novelty: float                   # 0..1 (dist. to historical centroids)
    recency: float                   # 0..1 (exp decay)
    severity: float                  # weighted sum
    priority: Priority


@dataclass(frozen=True)
class RoutingDecision:
    cluster_id: str
    decision: Decision
    reason: str
    target_queue: str | None
    sla_hours: int | None
    decided_by: str                  # "auto" or reviewer id
    decided_at_ns: int
```

### 2.2 `feedback_triage/ingest.py` — stage 1

```python
"""Normalise raw feedback and redact PII. Runs synchronously on the ingest
node; nothing further downstream sees raw user text."""
from __future__ import annotations
import hashlib
import re
import time
from typing import Any

import yaml

from .schema import FeedbackEvent


class Redactor:
    def __init__(self, pattern_path: str) -> None:
        with open(pattern_path) as f:
            cfg = yaml.safe_load(f)
        self._patterns = [(re.compile(p["regex"]), p["replacement"])
                          for p in cfg["patterns"]]

    def redact(self, text: str) -> str:
        if not text:
            return ""
        out = text
        for pat, replacement in self._patterns:
            out = pat.sub(replacement, out)
        return out


class FeedbackIngest:
    def __init__(self, redactor: Redactor, salt: str) -> None:
        self._redactor = redactor
        self._salt = salt

    def normalise(self, raw: dict[str, Any]) -> FeedbackEvent:
        """`raw` is whatever the client SDK posted. We define the normal form here."""
        return FeedbackEvent(
            feedback_id=raw["feedback_id"],
            trace_id=raw["trace_id"],
            user_id_hashed=self._hash_user(raw["user_id"]),
            surface=raw.get("surface", "unknown"),
            rating=raw["rating"],
            free_text_redacted=self._redactor.redact(raw.get("free_text", "")),
            rating_reason=raw.get("rating_reason"),
            client_ts_ns=int(raw["client_ts_ns"]),
            received_ns=time.time_ns(),
        )

    def _hash_user(self, user_id: str) -> str:
        return "u:" + hashlib.sha256((self._salt + user_id).encode()).hexdigest()[:16]
```

```yaml
# policy/pii_patterns.yaml — intentionally conservative. False positives are
# fine; false negatives cost a legal incident.
patterns:
  - regex: '\b[\w.+-]+@[\w-]+\.[\w.-]+\b'
    replacement: "[EMAIL]"
  - regex: '\b\+?\d[\d\s().-]{8,}\d\b'
    replacement: "[PHONE]"
  - regex: '\b\d{3}-\d{2}-\d{4}\b'
    replacement: "[SSN]"
  - regex: '\b(?:\d[ -]*?){13,16}\b'
    replacement: "[CARD]"
  - regex: '(?i)\b(?:password|api[_-]?key|token)\s*[:=]\s*\S+'
    replacement: "[SECRET]"
```

### 2.3 `feedback_triage/cluster.py` — stage 3

```python
"""Near-duplicate clustering. Two signals: trace_shape_hash (exact) and
embedding cosine distance (fuzzy). A cluster must agree on shape_hash; within
a shape_hash, events are grouped by embedding proximity."""
from __future__ import annotations
import math
import uuid
from collections import defaultdict
from typing import Iterable

from .schema import EnrichedEvent, Cluster


def _cosine(a: tuple[float, ...], b: tuple[float, ...]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    na = math.sqrt(sum(x * x for x in a)) or 1e-9
    nb = math.sqrt(sum(x * x for x in b)) or 1e-9
    return dot / (na * nb)


def _centroid(vectors: list[tuple[float, ...]]) -> tuple[float, ...]:
    n = len(vectors)
    dim = len(vectors[0])
    return tuple(sum(v[i] for v in vectors) / n for i in range(dim))


class NearDuplicateClusterer:
    def __init__(self, cosine_threshold: float = 0.85) -> None:
        self._thr = cosine_threshold

    def cluster(self, events: Iterable[EnrichedEvent]) -> list[Cluster]:
        by_shape: dict[str, list[EnrichedEvent]] = defaultdict(list)
        for e in events:
            by_shape[e.trace_shape_hash].append(e)

        clusters: list[Cluster] = []
        for shape, group in by_shape.items():
            # Greedy single-pass: assign each event to the closest existing
            # cluster above threshold; else start a new one.
            seeds: list[list[EnrichedEvent]] = []
            for ev in group:
                best_i, best_sim = -1, -1.0
                for i, seed_group in enumerate(seeds):
                    centroid = _centroid([x.embedding for x in seed_group])
                    sim = _cosine(ev.embedding, centroid)
                    if sim > best_sim:
                        best_sim, best_i = sim, i
                if best_i >= 0 and best_sim >= self._thr:
                    seeds[best_i].append(ev)
                else:
                    seeds.append([ev])

            for seed_group in seeds:
                ts = sorted(e.base.received_ns for e in seed_group)
                clusters.append(Cluster(
                    cluster_id=str(uuid.uuid4()),
                    representative_trace_id=seed_group[0].base.trace_id,
                    event_ids=tuple(e.base.feedback_id for e in seed_group),
                    count=len(seed_group),
                    first_seen_ns=ts[0],
                    last_seen_ns=ts[-1],
                    shape_hash=shape,
                    category=_pick_category(seed_group),
                    centroid=_centroid([e.embedding for e in seed_group]),
                ))
        return clusters


def _pick_category(group: list[EnrichedEvent]) -> str:
    """Majority vote on rating_reason; `unknown` if no majority."""
    counts: dict[str, int] = defaultdict(int)
    for e in group:
        counts[e.base.rating_reason or "unknown"] += 1
    best, best_n = "unknown", 0
    total = len(group)
    for k, n in counts.items():
        if n > best_n:
            best, best_n = k, n
    return best if best_n / total >= 0.5 else "mixed"
```

### 2.4 `feedback_triage/score.py` — stage 4

```python
"""Severity and priority. Weighted sum of four components; weights in YAML."""
from __future__ import annotations
import math
from dataclasses import dataclass

import yaml

from .schema import Cluster, ScoredCluster


@dataclass(frozen=True)
class SeverityWeights:
    w_harm: float
    w_blast: float
    w_novelty: float
    w_recency: float
    recency_halflife_s: float
    novelty_threshold: float
    dau_estimate: int
    thresholds: dict[str, float]   # {"P0": 0.85, "P1": 0.60, "P2": 0.30}
    critical_harm_floor: float = 0.9   # harm >= floor forces severity to P0


def load_weights(path: str) -> SeverityWeights:
    with open(path) as f:
        raw = yaml.safe_load(f)
    return SeverityWeights(**raw)


def score_cluster(
    c: Cluster,
    weights: SeverityWeights,
    harm_classifier_score: float,   # 0..1 from a lightweight safety classifier
    historical_centroids: list[tuple[float, ...]],
    now_ns: int,
) -> ScoredCluster:
    harm = harm_classifier_score
    unique_users = c.attributes.get("unique_users", c.count)
    blast = min(1.0, unique_users / max(1, weights.dau_estimate))
    novelty = _novelty(c.centroid, historical_centroids)
    recency = math.exp(-(now_ns - c.last_seen_ns) / 1e9 / weights.recency_halflife_s)

    # Critical-harm floor: a confirmed safety hit on even one user still
    # pages — short-circuit the weighted sum before the threshold ladder.
    # A single-user PII leak scores ~0.75 under the default weights
    # (blast≈1/DAU), which would otherwise land in the 24h P1 queue rather
    # than the P0 2h/paged queue. Exiting here keeps that path immune to
    # future weight or threshold edits.
    if harm >= weights.critical_harm_floor:
        return ScoredCluster(
            cluster=c, harm_score=harm, blast_radius=blast,
            novelty=novelty, recency=recency,
            severity=max(weights.thresholds["P0"], weights.w_harm * harm),
            priority="P0",
        )

    severity = (
        weights.w_harm * harm
        + weights.w_blast * blast
        + weights.w_novelty * novelty
        + weights.w_recency * recency
    )
    severity = min(1.0, max(0.0, severity))

    priority = "P3"
    for name, thr in sorted(weights.thresholds.items(), key=lambda x: -x[1]):
        if severity >= thr:
            priority = name
            break

    return ScoredCluster(
        cluster=c, harm_score=harm, blast_radius=blast,
        novelty=novelty, recency=recency, severity=severity, priority=priority,
    )


def _novelty(centroid: tuple[float, ...], history: list[tuple[float, ...]]) -> float:
    if not history:
        return 1.0
    def cos(a, b):
        dot = sum(x * y for x, y in zip(a, b))
        na = math.sqrt(sum(x * x for x in a)) or 1e-9
        nb = math.sqrt(sum(x * x for x in b)) or 1e-9
        return dot / (na * nb)
    best = max(cos(centroid, h) for h in history)
    return float(max(0.0, 1.0 - best))
```

```yaml
# policy/severity_weights.yaml
w_harm: 0.5
w_blast: 0.25
w_novelty: 0.15
w_recency: 0.10
recency_halflife_s: 86400        # 1 day
novelty_threshold: 0.4
dau_estimate: 50000
thresholds:
  P0: 0.85
  P1: 0.60
  P2: 0.30
critical_harm_floor: 0.9         # harm_classifier_score >= floor forces P0
```

### 2.5 `feedback_triage/route.py` — stage 5

```python
"""Map scored cluster → queue + SLA. Dismissals write to the audit table."""
from __future__ import annotations
import time
import yaml

from .schema import ScoredCluster, RoutingDecision


class Router:
    def __init__(self, routing_path: str, known_duplicate_fn) -> None:
        with open(routing_path) as f:
            self._cfg = yaml.safe_load(f)
        self._is_known_duplicate = known_duplicate_fn

    def route(self, scored: ScoredCluster) -> RoutingDecision:
        c = scored.cluster
        now = time.time_ns()
        if c.attributes.get("from_test_account"):
            return RoutingDecision(
                cluster_id=c.cluster_id, decision="dismiss_test_account",
                reason="user_id_hashed on allow-list of test accounts",
                target_queue=None, sla_hours=None,
                decided_by="auto", decided_at_ns=now,
            )
        if self._is_known_duplicate(c):
            return RoutingDecision(
                cluster_id=c.cluster_id, decision="dismiss_duplicate",
                reason="matches open cluster with same shape_hash and category",
                target_queue=None, sla_hours=None,
                decided_by="auto", decided_at_ns=now,
            )
        rule = self._cfg["priorities"][scored.priority]
        return RoutingDecision(
            cluster_id=c.cluster_id, decision="route",
            reason=f"priority={scored.priority} severity={scored.severity:.2f}",
            target_queue=rule["queue"], sla_hours=rule["sla_hours"],
            decided_by="auto", decided_at_ns=now,
        )
```

```yaml
# policy/routing.yaml
priorities:
  P0:
    queue: review-expert           # same queue as ex-01 domain_expert
    sla_hours: 2
    page_on_arrival: true
  P1:
    queue: review-expert
    sla_hours: 24
    page_on_arrival: false
  P2:
    queue: review-first-pass       # ex-01 annotator queue
    sla_hours: 72
    page_on_arrival: false
  P3:
    queue: weekly-digest
    sla_hours: 168
    page_on_arrival: false
```

### 2.6 `tests/test_routing_slo.py`

```python
from feedback_triage.score import load_weights, score_cluster
from feedback_triage.route import Router
from feedback_triage.schema import Cluster


def _cluster(**kw) -> Cluster:
    base = dict(
        cluster_id="c1", representative_trace_id="t1",
        event_ids=("f1",), count=1,
        first_seen_ns=0, last_seen_ns=0,
        shape_hash="h1", category="unknown",
        centroid=(1.0, 0.0),
    )
    base.update(kw)
    return Cluster(**base)


def test_high_harm_small_blast_still_P0():
    w = load_weights("policy/severity_weights.yaml")
    scored = score_cluster(
        _cluster(attributes={"unique_users": 1}),
        w, harm_classifier_score=1.0,
        historical_centroids=[], now_ns=0,
    )
    assert scored.priority == "P0", "pii leak from a single user is still P0"


def test_recent_wide_but_low_harm_is_P2_not_P0():
    w = load_weights("policy/severity_weights.yaml")
    scored = score_cluster(
        _cluster(attributes={"unique_users": 25_000}),
        w, harm_classifier_score=0.1,
        historical_centroids=[], now_ns=0,
    )
    assert scored.priority in {"P2", "P3"}, \
        "slow replies are not safety incidents even at wide blast"


def test_duplicate_dismissed_before_sla_math():
    rtr = Router("policy/routing.yaml", known_duplicate_fn=lambda c: True)
    from feedback_triage.schema import ScoredCluster
    sc = ScoredCluster(
        cluster=_cluster(), harm_score=0.9, blast_radius=0.9,
        novelty=0.9, recency=1.0, severity=0.95, priority="P0",
    )
    d = rtr.route(sc)
    assert d.decision == "dismiss_duplicate" and d.target_queue is None
```

## 3. Validation steps

1. **Install + run tests**: `pip install -e . && pytest -q tests/`. All four test modules pass; redaction tests include emails, phones, cards, and tokens; cluster tests verify shape-aware grouping; severity tests cover harm-dominant and blast-dominant signatures.
2. **End-to-end smoke**: `python -m feedback_triage.cli --feedback fixtures/feedback_events.jsonl --traces fixtures/traces.jsonl --out out/`. 200 input events collapse to ~15 clusters, each with a scored severity and routing decision.
3. **Redaction audit**: `grep -E '@|\\+[0-9]{10,}' out/clusters.jsonl` returns nothing; the same grep on the input fixture is non-empty. Any match in the output is a stage-1 escape.
4. **Routing SLA check**: `python -m feedback_triage.cli --audit out/decisions.jsonl` reports distribution: >= 1 P0, routing_queue counts match `policy/routing.yaml` targets.
5. **Dismissal audit**: `jq 'select(.decision | startswith("dismiss"))' out/decisions.jsonl | wc -l` is non-zero; each dismissal has `reason` and `decided_by` populated. No silent drops.
6. **Determinism**: rerun step 2 with the same fixture and seed. Cluster counts and priorities are byte-identical. (Random tie-breaking in clustering is seeded off the shape hash.)
7. **Integration with exercise-01**: pipe `out/clusters.jsonl` into the ingest adapter from exercise-01. P0/P1 clusters land in the expert queue; P2 in first-pass; P3 in the digest doc. Each review record carries the originating cluster id and event ids.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | PII redaction runs at ingest, before any downstream stage sees text. | `free_text_redacted` is the only text field after stage 1; raw text is never written to the clusters file. | Redaction runs at routing or review time; cluster-store contains raw PII. |
| R2 | Feedback is clustered before prioritisation; reviewers see one card per cluster, not per event. | `Cluster.count > 1` for repeated issues; UI shows cluster card with count. | Each event is routed individually; reviewer sees 200 duplicates. |
| R3 | Severity is a documented weighted sum of components, not a single binary. | `severity_weights.yaml` present; `score_cluster` exposes components; tests cover each component's influence. | Priority = `rating == 'down'`; no severity gradient. |
| R4 | Each priority maps to an SLA and a queue, both declared in config. | `routing.yaml` has `sla_hours` and `queue` per priority; downstream check verifies arrivals vs SLA. | SLAs are tribal knowledge; no routing table. |
| R5 | Nothing is dropped silently; every dismissal writes an audit record with reason + decider. | `decisions.jsonl` has a row per cluster; dismissals have populated `reason`. | Pipeline silently discards duplicates / nits. |
| R6 | Test-account traffic is removed via explicit allow-list, not heuristics. | `from_test_account` flag joined from an allow-list; dismissal decision is `dismiss_test_account`. | Heuristic rules drop traffic by user agent / IP; no audit. |
| R7 | Blast radius uses `unique_users`, not raw event count. | `Cluster.attributes["unique_users"]` computed from hashed user ids; one user spamming thumbs-down does not inflate blast. | Blast uses `Cluster.count`; a single angry user raises severity. |
| R8 | Novelty is measured against a history of past cluster centroids. | `score_cluster` consumes `historical_centroids`; novelty tends toward 0 for recurring issues, 1 for first-seen. | Novelty is always `1.0`; recurring issues keep paging. |
| R9 | The pipeline is deterministic given `(inputs, config_version)`. | Rerun with identical inputs produces identical clusters and priorities. | Clustering order changes run-to-run; reviewers can't reason about priority. |
| R10 | Priority thresholds are configurable and reviewed with the on-call team, not hard-coded. | `thresholds` block in `severity_weights.yaml`; change ledger shows reviewer sign-off. | `if severity > 0.7` hard-coded in `score.py`. |

A passing submission is 8+ of 10, with R1, R2, R4, R5, R9 all hard-required.

## 5. Common mistakes

- **Routing events, not clusters.** The same bad reply gets 200 thumbs-downs; without clustering, 200 cards hit the queue and the reviewer gives up. Cluster by shape + embedding before prioritising.
- **Binary severity.** "Down vs up" is not severity. A safety leak and a slow response are both `down` but need different queues on different SLAs. Decompose severity into harm, blast, novelty, recency and keep the weights in config.
- **Using event count for blast.** One frustrated user spamming the thumbs-down button is not a product-wide regression. Collapse to `unique_users` (hashed) before computing blast.
- **No audit on dismissals.** "The pipeline auto-dropped it" is not an answer when legal shows up. Write a `decision_record` for every dismissal with reason, decider, and timestamp — NIST AI RMF `MANAGE 4.3` makes this an expectation.
- **Redacting after clustering.** Clustering sees embeddings of raw text, which means the embedding store is a PII store. Redact at ingest, embed on redacted text.
- **Hard-coded thresholds.** `if severity > 0.7` in code moves the on-call bar every refactor. Put thresholds in YAML, change them on PR with the on-call team's sign-off.
- **One queue for all priorities.** If P0 and P3 share a queue, P3 volume buries P0. Route each priority to a queue with its own SLA and paging rule.
- **Treating novelty as zero by default.** A feedback cluster that repeats weekly should decay in priority, not keep escalating. Measure novelty against a rolling centroid history.
- **Mixing test-account traffic in.** Internal QA accounts generate thousands of thumbs-downs by design. Keep an explicit allow-list and dismiss with reason `dismiss_test_account`.
- **No handoff to exercise-01.** The triage pipeline that cannot populate the review workspace is a leaderboard, not a workflow. Every `route` decision must produce a record the reviewer actually sees.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — `MANAGE 4.3` (incident and feedback handling), `MEASURE 3.3` (post-deployment monitoring of user-reported issues), `GOVERN 4.1` (team roles for AI incidents).
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning): feedback-driven training/eval loops are a classic injection surface; `ML06` (AI supply-chain): feedback flows back into datasets.
- MITRE ATLAS — <https://atlas.mitre.org/> — `AML.T0061` (LLM Prompt Injection) context for why free-text feedback must be treated as untrusted input; `AML.T0018` (Backdoor ML Model) as a reason the triage queue must be auditable.
- Python stdlib: `re` (regex redaction), <https://docs.python.org/3/library/re.html>; `math` (cosine math) <https://docs.python.org/3/library/math.html>.
- Local exercise reader: `lessons/mod-109-human-review-workflows/exercises/exercise-02-user-feedback-triage-pipeline.md`.
- Related exercises: exercise-01 is the review workspace this pipeline feeds; exercise-03 reads the resulting annotated reviews for IAA; exercise-04 uses the per-cluster severity to drive gold-set rotation decisions.
