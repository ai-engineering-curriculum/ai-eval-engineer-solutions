# exercise-04 — Gold Set Rotation With Surface Change — Solution

> Reference solution for `lessons/mod-109-human-review-workflows/exercises/exercise-04-gold-set-rotation-with-surface-change.md`.
> Design the process + tooling for rotating the gold set when the product surface changes (new tool, new UI, new policy), so that the reviewed-against-gold accuracy signal from exercise-03 remains valid and the regression suite from mod-106 does not quietly drift from reality.

## 1. Solution overview

The deliverable is a rotation policy + a small toolchain (`gold_rotator/`) plus a change-control document template. The core problem: when the product changes, the gold set from last quarter starts measuring something that is no longer the product. If you keep scoring against it, you get a stable number that says nothing; if you throw it out, you lose continuity. The rotation is the controlled reconciliation between those failure modes.

The operating idea is **bands, not full rotation**. The gold set is partitioned into three bands:

- **Anchor** (~40%): surface-agnostic items whose correct answer is unaffected by the surface change (e.g., refusing a prompt-injection; answering a factual question with grounded context). These persist across rotations.
- **Surface-coupled** (~50%): items whose correct answer depends on the surface being changed (specific tool arguments, policy phrasing, UI affordances). These are the retire/replace candidates on a surface change.
- **Canary** (~10%): freshly drawn, recently-adjudicated items used to detect regressions early. These rotate every two weeks regardless of surface changes.

Five design choices anchor the solution:

1. **Classify items at ingest, not during the rotation.** Each gold item carries a `surface_dependency` tag (set when the adjudicator adds it) listing which surfaces its correctness depends on. When a surface changes, the rotator filters by that tag; no human has to reconsider the whole set.
2. **Rotations are change-controlled, not ambient.** A surface change produces a Rotation RFC with: what changed, which items are candidates for retire / replace / anchor, who signs off, and what the before/after IAA baseline is. Rotations do not happen because someone noticed on a Thursday.
3. **Retired items are archived, not deleted.** The gold-set table has a `status` column (`active`, `retired_r<n>`, `canary`); retired items stay queryable for historical score replay. Deletion breaks mod-106's eval-gated CI retroactively and is irreversible.
4. **Replacement candidates come from exercise-02's P0/P1 adjudicated clusters.** When we retire an item, the replacement is pulled from a pool of adjudicated high-priority clusters; this keeps the gold set aligned to real user harm, not synthetic edge cases a test author imagined.
5. **A paired-score guard prevents regressions from being masked by rotation.** When we publish a new gold set version, the harness runs the *same* model on both the old active gold and the new active gold and reports both scores for one overlap period (two weeks). A big delta on anchor items is a smell — either the model regressed, or the "anchor" classification was wrong.

## 2. Worked implementation

```
mod-109/exercise-04/
├── pyproject.toml
├── README.md
├── gold_rotator/
│   ├── __init__.py
│   ├── schema.py             # GoldItem, SurfaceChange, RotationRFC
│   ├── store.py              # SQLite-backed gold set with status history
│   ├── classify.py           # anchor / surface-coupled / canary classifier helpers
│   ├── rotation.py           # execute a rotation plan
│   ├── paired_score.py       # run eval against old+new gold during overlap window
│   ├── rfc.py                # Rotation RFC renderer
│   └── cli.py
├── policy/
│   ├── band_targets.yaml     # target percentages + rotation cadence
│   └── surface_registry.yaml # known surfaces + their current version
├── templates/
│   └── ROTATION_RFC.md.tmpl
├── fixtures/
│   ├── gold_v3.jsonl
│   └── adjudicated_clusters.jsonl
└── tests/
    ├── test_band_targets.py
    ├── test_rotation_preserves_anchors.py
    ├── test_paired_score_overlap.py
    └── test_rfc_blocks_without_signoff.py
```

### 2.1 `gold_rotator/schema.py`

```python
"""Gold-set and rotation dataclasses."""
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal

Band = Literal["anchor", "surface_coupled", "canary"]
Status = Literal["active", "retired", "pending"]


@dataclass(frozen=True)
class GoldItem:
    gold_id: str                     # stable across rotations
    trace_id: str                    # originating trace (from mod-107 ex-01 sink)
    rubric_version: str              # rubric that adjudicated this item
    adjudicated_values: dict         # ground-truth labels (field_id -> value)
    adjudicator_id: str
    adjudicated_at_ns: int
    band: Band
    surface_dependency: tuple[str, ...]   # e.g. ("checkout_tool_v2", "safety_policy_v3")
    status: Status
    rotation_version: int            # which gold-set version introduced this item
    retired_at_ns: int | None = None
    retired_reason: str | None = None


@dataclass(frozen=True)
class SurfaceChange:
    surface_id: str                  # e.g. "checkout_tool"
    from_version: str                # e.g. "v2"
    to_version: str
    change_summary: str              # short free-text, becomes RFC title
    changed_at_ns: int


@dataclass(frozen=True)
class RotationPlan:
    rfc_id: str
    surface_change: SurfaceChange
    retire: tuple[str, ...]          # gold_ids to retire
    add: tuple[str, ...]             # trace_ids to promote from adjudicated_clusters
    anchor_moves: tuple[tuple[str, Band], ...]   # reclassifications
    reviewers: tuple[str, ...]       # names who must sign off


@dataclass(frozen=True)
class RotationRFC:
    plan: RotationPlan
    before_counts: dict[Band, int]
    after_counts: dict[Band, int]
    baseline_metrics: dict           # eval metrics on old gold (pre-rotation)
    signoffs: tuple[tuple[str, int], ...] = field(default_factory=tuple)
```

### 2.2 `gold_rotator/store.py`

```python
"""SQLite-backed gold set. Status history is append-only; retire is a state
transition, not a delete."""
from __future__ import annotations
import sqlite3
import time
from typing import Iterable

from .schema import GoldItem


DDL = """
CREATE TABLE IF NOT EXISTS gold_item (
    gold_id              TEXT PRIMARY KEY,
    trace_id             TEXT NOT NULL,
    rubric_version       TEXT NOT NULL,
    adjudicated_values   TEXT NOT NULL,
    adjudicator_id       TEXT NOT NULL,
    adjudicated_at_ns    INTEGER NOT NULL,
    band                 TEXT NOT NULL,
    surface_dependency   TEXT NOT NULL,
    status               TEXT NOT NULL,
    rotation_version     INTEGER NOT NULL,
    retired_at_ns        INTEGER,
    retired_reason       TEXT
);

CREATE TABLE IF NOT EXISTS status_log (
    gold_id      TEXT NOT NULL,
    at_ns        INTEGER NOT NULL,
    from_status  TEXT NOT NULL,
    to_status    TEXT NOT NULL,
    reason       TEXT NOT NULL,
    actor        TEXT NOT NULL,
    PRIMARY KEY (gold_id, at_ns)
);
"""


class GoldStore:
    def __init__(self, dsn: str) -> None:
        self._conn = sqlite3.connect(dsn)
        self._conn.executescript(DDL)

    def add(self, item: GoldItem) -> None:
        import json
        self._conn.execute(
            "INSERT INTO gold_item VALUES (?,?,?,?,?,?,?,?,?,?,?,?)",
            (
                item.gold_id, item.trace_id, item.rubric_version,
                json.dumps(item.adjudicated_values, sort_keys=True),
                item.adjudicator_id, item.adjudicated_at_ns,
                item.band, json.dumps(item.surface_dependency),
                item.status, item.rotation_version,
                item.retired_at_ns, item.retired_reason,
            ),
        )
        self._conn.commit()

    def retire(self, gold_id: str, actor: str, reason: str) -> None:
        now = time.time_ns()
        row = self._conn.execute(
            "SELECT status FROM gold_item WHERE gold_id = ?", (gold_id,)
        ).fetchone()
        if row is None or row[0] != "active":
            raise ValueError(f"cannot retire {gold_id}: status is {row and row[0]}")
        self._conn.execute(
            "UPDATE gold_item SET status='retired', retired_at_ns=?, retired_reason=? WHERE gold_id=?",
            (now, reason, gold_id),
        )
        self._conn.execute(
            "INSERT INTO status_log VALUES (?,?,?,?,?,?)",
            (gold_id, now, "active", "retired", reason, actor),
        )
        self._conn.commit()

    def reclassify(self, gold_id: str, new_band: str, actor: str, reason: str) -> None:
        now = time.time_ns()
        self._conn.execute(
            "UPDATE gold_item SET band=? WHERE gold_id=? AND status='active'",
            (new_band, gold_id),
        )
        self._conn.execute(
            "INSERT INTO status_log VALUES (?,?,?,?,?,?)",
            (gold_id, now, "active", "active", f"band:{new_band}:{reason}", actor),
        )
        self._conn.commit()

    def active_counts_by_band(self) -> dict[str, int]:
        rows = self._conn.execute(
            "SELECT band, COUNT(*) FROM gold_item WHERE status='active' GROUP BY band"
        ).fetchall()
        return dict(rows)
```

### 2.3 `gold_rotator/rotation.py`

```python
"""Execute a rotation plan. Validates band targets, runs the retire/add
operations, and refuses to commit without the required sign-offs."""
from __future__ import annotations
import json
import time
import uuid
from pathlib import Path

import yaml

from .schema import GoldItem, RotationPlan, SurfaceChange
from .store import GoldStore


class RotationError(Exception):
    pass


def load_band_targets(path: str) -> dict:
    with open(path) as f:
        return yaml.safe_load(f)


def plan_rotation(
    store: GoldStore,
    change: SurfaceChange,
    candidate_clusters: list[dict],
    targets: dict,
    reviewers: list[str],
) -> RotationPlan:
    rows = store._conn.execute(
        "SELECT gold_id, band, surface_dependency, status FROM gold_item"
    ).fetchall()
    retire: list[str] = []
    for gold_id, band, dep_json, status in rows:
        if status != "active":
            continue
        deps = json.loads(dep_json)
        tagged = f"{change.surface_id}_{change.from_version}"
        if tagged in deps and band == "surface_coupled":
            retire.append(gold_id)

    # Choose replacements from adjudicated clusters: oldest P0/P1 first.
    replacements = [c["representative_trace_id"]
                    for c in candidate_clusters
                    if c["priority"] in ("P0", "P1")][: len(retire) + 5]

    return RotationPlan(
        rfc_id=f"rfc-{uuid.uuid4().hex[:8]}",
        surface_change=change,
        retire=tuple(retire),
        add=tuple(replacements),
        anchor_moves=(),  # populated by reviewer during RFC review
        reviewers=tuple(reviewers),
    )


def execute_rotation(
    store: GoldStore,
    plan: RotationPlan,
    adjudicated_items: dict[str, GoldItem],
    signoffs: list[tuple[str, int]],
    targets: dict,
) -> None:
    required = set(plan.reviewers)
    received = {name for name, _ in signoffs}
    if not required.issubset(received):
        missing = required - received
        raise RotationError(f"missing signoffs: {sorted(missing)}")

    for gold_id in plan.retire:
        store.retire(gold_id, actor="rotator", reason=f"{plan.rfc_id}:surface_change")

    for trace_id in plan.add:
        item = adjudicated_items[trace_id]
        store.add(item)

    for gold_id, new_band in plan.anchor_moves:
        store.reclassify(gold_id, new_band, actor="rotator", reason=plan.rfc_id)

    _validate_band_ratios(store, targets)


def _validate_band_ratios(store: GoldStore, targets: dict) -> None:
    counts = store.active_counts_by_band()
    total = sum(counts.values()) or 1
    for band, t in targets["bands"].items():
        pct = counts.get(band, 0) / total
        lo, hi = t["min_ratio"], t["max_ratio"]
        if not (lo <= pct <= hi):
            raise RotationError(
                f"after rotation, {band} ratio {pct:.2f} outside [{lo}, {hi}]"
            )
```

```yaml
# policy/band_targets.yaml
bands:
  anchor:
    min_ratio: 0.35
    max_ratio: 0.50
  surface_coupled:
    min_ratio: 0.40
    max_ratio: 0.60
  canary:
    min_ratio: 0.05
    max_ratio: 0.15
cadence:
  canary_days: 14              # canary rotates every 2 weeks regardless
  full_review_days: 90         # whole set re-reviewed every 90 days
  surface_change_cooldown_days: 7  # min delay between consecutive rotations
required_signoffs:
  - ai-eval-lead
  - product-owner-for-surface
```

### 2.4 `gold_rotator/paired_score.py`

```python
"""Run the eval harness against both the old and new gold during the
overlap window; report both metrics to catch masked regressions."""
from __future__ import annotations
from dataclasses import dataclass
from typing import Callable


@dataclass
class PairedScoreReport:
    rotation_version: int
    on_old_gold: dict[str, float]     # metric_name -> score
    on_new_gold: dict[str, float]
    anchor_delta: float
    smell: str | None                 # populated if the anchor delta is suspicious


def run_paired_score(
    candidate_eval_fn: Callable[[list[dict]], dict[str, float]],
    old_gold_items: list[dict],
    new_gold_items: list[dict],
    anchor_tolerance: float = 0.03,
) -> PairedScoreReport:
    on_old = candidate_eval_fn(old_gold_items)
    on_new = candidate_eval_fn(new_gold_items)

    old_anchors = [g for g in old_gold_items if g["band"] == "anchor"]
    new_anchors = [g for g in new_gold_items if g["band"] == "anchor"]
    anchor_old_score = candidate_eval_fn(old_anchors).get("accuracy", float("nan"))
    anchor_new_score = candidate_eval_fn(new_anchors).get("accuracy", float("nan"))
    delta = anchor_new_score - anchor_old_score

    smell = None
    if abs(delta) > anchor_tolerance:
        smell = (
            f"anchor delta {delta:+.3f} exceeds tolerance {anchor_tolerance}; "
            "either the model regressed on surface-agnostic items or an "
            "anchor item was misclassified and secretly surface-coupled."
        )

    return PairedScoreReport(
        rotation_version=new_gold_items[0]["rotation_version"],
        on_old_gold=on_old, on_new_gold=on_new,
        anchor_delta=delta, smell=smell,
    )
```

### 2.5 `templates/ROTATION_RFC.md.tmpl`

```markdown
# Gold Set Rotation RFC — {{ rfc_id }}

**Trigger:** {{ surface_change.surface_id }} {{ surface_change.from_version }} → {{ surface_change.to_version }}
**Summary:** {{ surface_change.change_summary }}
**Proposed rotation version:** v{{ new_version }}
**Required sign-offs:** {{ reviewers | join(", ") }}

## 1. Scope of change

[one paragraph: what in the product changed, what user-visible behaviour this
alters, which safety/policy surface moved]

## 2. Impact on current gold set

Before rotation (active items):

| Band | Count | % |
|------|-------|---|
| anchor | {{ before.anchor }} | {{ before.anchor_pct }} |
| surface_coupled | {{ before.surface_coupled }} | {{ before.surface_coupled_pct }} |
| canary | {{ before.canary }} | {{ before.canary_pct }} |

Proposed retires: {{ retire | length }} items (all `surface_coupled` depending on
{{ surface_change.surface_id }}_{{ surface_change.from_version }}).

Proposed additions: {{ add | length }} items from adjudicated clusters
(exercise-02 P0/P1 last 60 days).

After rotation (projected):

| Band | Count | % |
|------|-------|---|
| anchor | {{ after.anchor }} | {{ after.anchor_pct }} |
| surface_coupled | {{ after.surface_coupled }} | {{ after.surface_coupled_pct }} |
| canary | {{ after.canary }} | {{ after.canary_pct }} |

Band ratios must fall inside `policy/band_targets.yaml` — this RFC will
not commit if they do not.

## 3. Baseline metric

Model `{{ model_id }}` on current active gold: `{{ baseline.accuracy }}`.
This is the number the next paired-score report compares against.

## 4. Paired-score plan

During the 14-day overlap window, CI runs the candidate model against
both old-active and new-active gold. The paired-score report flags any
`anchor_delta` outside ±0.03 as a potential masked regression or a
misclassified anchor.

## 5. Sign-offs

- [ ] {{ reviewers[0] }}
- [ ] {{ reviewers[1] }}
...

## 6. Retire + add manifests

<details>
<summary>Retired gold_ids (click to expand)</summary>

```
{{ retire | join("\n") }}
```

</details>

<details>
<summary>Added trace_ids</summary>

```
{{ add | join("\n") }}
```

</details>
```

### 2.6 `tests/test_rotation_preserves_anchors.py`

```python
import json
import time
from gold_rotator.schema import GoldItem, SurfaceChange
from gold_rotator.rotation import plan_rotation, execute_rotation, load_band_targets
from gold_rotator.store import GoldStore


def _item(gold_id, band, deps, status="active") -> GoldItem:
    return GoldItem(
        gold_id=gold_id, trace_id=f"t_{gold_id}",
        rubric_version="v1.0", adjudicated_values={"helpfulness": "yes"},
        adjudicator_id="adj1", adjudicated_at_ns=0,
        band=band, surface_dependency=tuple(deps),
        status=status, rotation_version=3,
    )


def test_anchor_items_are_not_retired_on_surface_change(tmp_path):
    store = GoldStore(str(tmp_path / "gold.db"))
    store.add(_item("g1", "anchor", ["checkout_tool_v2"]))
    store.add(_item("g2", "surface_coupled", ["checkout_tool_v2"]))
    store.add(_item("g3", "surface_coupled", ["safety_policy_v3"]))

    change = SurfaceChange(
        surface_id="checkout_tool", from_version="v2", to_version="v3",
        change_summary="rewrite arg shape",
        changed_at_ns=time.time_ns(),
    )
    candidates = [
        {"representative_trace_id": "new_t1", "priority": "P0"},
        {"representative_trace_id": "new_t2", "priority": "P1"},
    ]
    plan = plan_rotation(store, change, candidates,
                         load_band_targets("policy/band_targets.yaml"),
                         reviewers=["ai-eval-lead", "product-owner-for-surface"])

    # anchor stays; surface-coupled matching the change goes.
    assert "g1" not in plan.retire
    assert "g2" in plan.retire
    assert "g3" not in plan.retire   # different surface
```

### 2.7 `tests/test_rfc_blocks_without_signoff.py`

```python
import pytest
from gold_rotator.rotation import execute_rotation, RotationError
from gold_rotator.schema import RotationPlan, SurfaceChange


def test_execute_refuses_without_required_signoffs(tmp_path):
    from gold_rotator.store import GoldStore
    store = GoldStore(str(tmp_path / "gold.db"))
    plan = RotationPlan(
        rfc_id="rfc-test", surface_change=SurfaceChange(
            surface_id="x", from_version="v1", to_version="v2",
            change_summary="t", changed_at_ns=0,
        ),
        retire=(), add=(),
        anchor_moves=(),
        reviewers=("ai-eval-lead", "product-owner-for-surface"),
    )
    with pytest.raises(RotationError, match="missing signoffs"):
        execute_rotation(store, plan, adjudicated_items={},
                         signoffs=[("ai-eval-lead", 1)], targets={})
```

## 3. Validation steps

1. **Install + run tests**: `pip install -e . && pytest -q tests/`. All four test modules pass; the sign-off test specifically verifies the rotator refuses to commit with partial approvals.
2. **End-to-end**: `python -m gold_rotator.cli plan --surface checkout_tool --from v2 --to v3 --store fixtures/gold_v3.jsonl --clusters fixtures/adjudicated_clusters.jsonl --out out/`. Writes `out/rotation_plan.json`, `out/ROTATION_RFC.md`.
3. **Human sign-off path**: inspect `out/ROTATION_RFC.md`; it shows before/after counts, the retire manifest, the add manifest, and empty checkboxes next to the reviewer names. The rotator exits 0 but does not commit.
4. **Commit with sign-offs**: `python -m gold_rotator.cli commit --rfc out/ROTATION_RFC.md --signoff "ai-eval-lead:$(whoami)" --signoff "product-owner-for-surface:$(whoami)"`. The store now has retired rows (`status='retired'`) and new active rows. Band ratios fall inside `policy/band_targets.yaml`.
5. **Historical replay**: `python -m gold_rotator.cli replay --version 3`. The harness reconstructs the v3 active set (retired items re-included if their retire ts is > v3's cutoff). mod-106's CI can still run the v3 evaluation.
6. **Paired-score during overlap**: `python -m gold_rotator.paired_score --old v3 --new v4 --model model-xyz`. Report shows separate accuracies on both gold sets; `anchor_delta` is near zero in the smoke fixture. Forcing a regression (edit one anchor to a wrong adjudicated value) makes the smell line fire.
7. **Canary cadence**: with `cadence.canary_days: 14`, re-running the planner with no surface change proposes rotating the ~10% canary items whose `added_at` is older than 14 days. Anchor items are untouched.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Gold items are classified into bands (anchor / surface-coupled / canary) with explicit `surface_dependency`. | Every item has non-null `band` + list `surface_dependency`; UI shows these. | Items are a flat list; no way to filter by surface dependency. |
| R2 | Rotations are triggered by a `SurfaceChange` record or a cadence timer, not by ambient decisions. | `cli plan` consumes a `SurfaceChange` or cadence trigger; log shows provenance. | Humans decide ad-hoc when to rotate. |
| R3 | Rotations are gated on named sign-offs; execution refuses without them. | `execute_rotation` raises `RotationError("missing signoffs")` test passes. | A single person can rotate silently. |
| R4 | Band ratios after rotation are validated against targets; rotation rolls back if out of range. | `_validate_band_ratios` raises; post-commit state is rejected. | Rotation can leave the canary at 0% or anchors at 10%. |
| R5 | Retired items are preserved and queryable for historical replay. | `status='retired'` kept in table; replay of a prior rotation reconstructs that version. | Retired items are deleted; past CI results cannot be reproduced. |
| R6 | A paired-score overlap window compares old-gold vs new-gold scores. | `paired_score` produces both numbers; anchor delta is explicit. | New gold replaces old immediately; a regression that happens with the rotation is undetectable. |
| R7 | Replacement items come from adjudicated clusters with provenance (exercise-02 priority tag + exercise-03 adjudicator). | Each added item carries `adjudicator_id`, `adjudicated_at_ns`, origin cluster/priority. | Replacements are hand-written by the engineer doing the rotation. |
| R8 | A rotation RFC is generated before commit and stored with sign-offs and before/after counts. | `ROTATION_RFC.md` written; sign-offs required to commit. | Rotation happens from a shell one-liner; no RFC. |
| R9 | Anchor reclassification requires reviewer action, not an automated rule. | `anchor_moves` is empty by default; must be explicitly set in the RFC. | Classifier moves items between bands without review. |
| R10 | The whole rotation (plan → RFC → sign-off → commit → paired-score) runs under one CLI, not notebook glue. | `gold_rotator.cli` provides all four verbs; deterministic. | Multi-tool manual process; nobody can replay a rotation. |

A passing submission is 8+ of 10, with R2, R3, R5, R6, R8 all hard-required.

## 5. Common mistakes

- **Delete-and-replace rotations.** Deleting retired items breaks replay. mod-106's CI runs against the gold set active at a given commit; if the retired items are gone, old CI runs are unreproducible. Keep retired items in the table with a status transition.
- **No bands.** Rotating "the gold set" as a single blob means every surface change either resets the whole suite (lose continuity) or ignores the change (lose validity). Bands make the trade-off explicit.
- **No paired-score overlap.** The worst regression is one that appears exactly at the rotation boundary: new gold comes in, score moves, nobody knows if it was the rotation or the model. A two-week overlap with paired scoring catches this.
- **Hand-writing replacement items.** Synthetic replacements drift toward what the author imagines the user does, not what users actually do. Pull replacements from exercise-02's P0/P1 adjudicated clusters so the gold set tracks real harm.
- **Ambient rotations.** "Someone noticed a cluster, so they added it to gold." This is how the gold set becomes arbitrary. Rotations go through an RFC with sign-offs; the RFC is a reviewable artefact.
- **No sign-off requirement.** If one engineer can rotate, the gold set is one engineer's opinion. NIST AI RMF `GOVERN 1.2` and `GOVERN 1.5` explicitly cover separation of duties for AI measurement.
- **Rotating anchors by mistake.** An item marked "anchor" that is actually surface-coupled will quietly skew the paired-score. If `anchor_delta` is > a few points, treat it as a classification bug first, model regression second.
- **Not rotating the canary.** The canary band exists to catch freshly emerging failure modes. Leaving it at zero or stale items turns it into decoration; rotate on a two-week cadence regardless of surface changes.
- **Rotating too often.** Rotations are expensive and reset continuity. The policy has a 7-day cooldown between consecutive rotations; respect it. If you need to rotate twice in a week, you probably need to bump the rubric version instead.
- **Decoupling from mod-106's CI.** The gold set that CI reads must be a specific rotation version, not "latest". Otherwise, a rotation silently changes CI without the PR that committed it.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — `GOVERN 1.2` (role assignment), `GOVERN 1.5` (separation of duties), `MEASURE 2.6` (periodic re-measurement of deployed AI systems).
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning) and `ML06` (AI supply chain): the gold set is a high-value target; rotation + sign-off is a control.
- MITRE ATLAS — <https://atlas.mitre.org/> — `AML.T0018` (Backdoor ML Model) and `AML.T0059` (Erode ML Model Integrity): rotation without audit trails is one of the attack preconditions listed; the rotation RFC + preserved history is a direct mitigation.
- Python stdlib: `sqlite3`, `uuid` — <https://docs.python.org/3/library/sqlite3.html>.
- Local exercise reader: `lessons/mod-109-human-review-workflows/exercises/exercise-04-gold-set-rotation-with-surface-change.md`.
- Related exercises: exercise-01's expert workspace writes adjudicated items; exercise-02's P0/P1 clusters are the replacement pool; exercise-03's IAA against gold is only valid for the active rotation version; mod-106's CI pins the active version as a commit-scoped artefact.
