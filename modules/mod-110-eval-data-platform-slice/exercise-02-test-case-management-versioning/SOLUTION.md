# exercise-02 — Test-Case Management and Versioning — Solution

> Reference solution for `lessons/mod-110-eval-data-platform-slice/exercises/exercise-02-test-case-management-versioning.md`.
> The deliverable is a system for managing eval test cases (golden sets) and their versions: a content-addressed store, a stable per-case identity across versions, an explicit lifecycle (draft → candidate → active → retired), and provenance so every verdict can be joined back to the exact dataset snapshot it was computed against.

## 1. Solution overview

The test-case plane has three fact-shaped tables and a release model built on top of them. Five design decisions anchor the solution:

1. **A test case has a stable identity; a test-case *version* is immutable.** `case_id` never changes once published; `case_version_id` is a content hash. Edits don't mutate rows, they produce new versions. This lets a flaky-fix refactor and a regression-fix edit coexist in history.
2. **Datasets are content-addressed sets of case versions.** A `dataset_version` is a signed snapshot: an ordered list of `(case_id, case_version_id)` pairs, hashed. Two writers producing the same cases in the same order get the same `dataset_version_id`; that id is the FK that exercise-01's `eval_result.dataset_version_id` points at.
3. **Lifecycle is explicit.** Cases move `draft → candidate → active → retired`. Only `active` cases participate in CI gates (exercise-05). `candidate` cases run in shadow — scored but not blocking. `retired` cases stay readable (for historical regression queries) but no new dataset can include them.
4. **Every change leaves an author and a reason.** A case version row carries `author`, `change_kind` (`fix`, `drift_update`, `new`, `retire`), and a `rationale` string. This is the audit trail the governance side of [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework) `GOVERN 2.1` requires, and the thing that prevents the "who changed the gold answer last Friday?" fire drill.
5. **Provenance for labels.** A test case's `expected` field has its own provenance: `label_source` (`human`, `judge`, `auto`), `annotator_id_hash`, `confidence`, and `rubric_version`. A single golden-set row is useless without this because labels age differently from inputs.

The result is a plane where every verdict can answer "which exact case, which exact version, under which exact rubric".

```
┌────────────────┐     ┌────────────────┐     ┌────────────────────┐
│    case        │     │  case_version  │     │  dataset_version   │
│ (stable id)    │ 1─N │ (immutable)    │ N─N │ (content-addressed)│
└────────────────┘     └────────────────┘     └────────────────────┘
        │                      │                       │
        │                      ▼                       │
        │             ┌────────────────┐               │
        │             │  label_source  │               │
        │             │ (provenance)   │               │
        │             └────────────────┘               │
        └────────► test_suite (release cut) ◄──────────┘
```

## 2. Worked implementation

### 2.1 `schemas/case.sql` — the three tables

```sql
-- case: stable identity. One row per "test case" concept. You rename, split,
-- or deprecate a case here. The case_id is permanent.
CREATE TABLE "case" (
    case_id             TEXT        PRIMARY KEY,          -- e.g. "support.order_lookup.happy_path.001"
    surface             TEXT        NOT NULL,             -- which app surface this case targets
    tags                TEXT[]      NOT NULL DEFAULT '{}', -- freeform taxonomy (e.g. ["rag", "high_risk"])
    created_ns          BIGINT      NOT NULL,
    created_by          TEXT        NOT NULL
);

-- case_version: immutable. One row per content snapshot. Content-addressed
-- by SHA of (input || expected || rubric_version). Edits make new rows,
-- never mutate existing ones.
CREATE TABLE case_version (
    case_version_id     TEXT        PRIMARY KEY,          -- "sha256:<hex>"
    case_id             TEXT        NOT NULL REFERENCES "case"(case_id),
    parent_version_id   TEXT        REFERENCES case_version(case_version_id),
    input               JSONB       NOT NULL,             -- the prompt / state / tool inputs
    expected            JSONB       NOT NULL,             -- gold answer / predicate / rubric
    label_source        TEXT        NOT NULL,             -- human | judge | auto
    annotator_id_hash   CHAR(16),                         -- NULL for auto/judge-produced labels
    rubric_version      TEXT        NOT NULL,             -- rubric these labels were written against
    confidence          REAL        NOT NULL DEFAULT 1.0, -- [0,1] annotator-declared confidence
    content_sha256      CHAR(64)    NOT NULL UNIQUE,
    created_ns          BIGINT      NOT NULL,
    author              TEXT        NOT NULL,
    change_kind         TEXT        NOT NULL,             -- new | fix | drift_update | retire
    rationale           TEXT        NOT NULL              -- why this change exists
);

-- dataset_version: a signed snapshot. Owned by this exercise; referenced
-- from exercise-01's eval_result.dataset_version_id.
CREATE TABLE dataset_version (
    dataset_version_id      TEXT        PRIMARY KEY,      -- "sha256:<hex>"
    dataset_name            TEXT        NOT NULL,         -- e.g. "support.regression"
    manifest_sha256         CHAR(64)    NOT NULL UNIQUE,
    manifest                JSONB       NOT NULL,         -- ordered list of (case_id, case_version_id)
    row_count               INT         NOT NULL,
    parent_version_id       TEXT        REFERENCES dataset_version(dataset_version_id),
    created_ns              BIGINT      NOT NULL,
    created_by              TEXT        NOT NULL,
    change_notes            TEXT        NOT NULL          -- what changed vs. parent
);

-- case_status: lifecycle. The active row is `MAX(effective_ns)` per
-- (case_id, case_version_id).
CREATE TABLE case_status (
    case_id             TEXT        NOT NULL REFERENCES "case"(case_id),
    case_version_id     TEXT        NOT NULL REFERENCES case_version(case_version_id),
    status              TEXT        NOT NULL,             -- draft | candidate | active | retired
    effective_ns        BIGINT      NOT NULL,
    actor               TEXT        NOT NULL,
    PRIMARY KEY (case_id, case_version_id, effective_ns)
);

CREATE INDEX case_version_by_case      ON case_version (case_id, created_ns DESC);
CREATE INDEX case_status_latest        ON case_status (case_id, case_version_id, effective_ns DESC);
```

### 2.2 `schemas/views.sql` — convenience read views

```sql
-- v_case_current_status: the current status of each (case_id, case_version_id) pair.
CREATE VIEW v_case_current_status AS
SELECT DISTINCT ON (case_id, case_version_id)
    case_id, case_version_id, status, effective_ns, actor
FROM case_status
ORDER BY case_id, case_version_id, effective_ns DESC;

-- v_active_cases: the set of cases eligible for an active dataset.
CREATE VIEW v_active_cases AS
SELECT cv.*
FROM case_version cv
JOIN v_case_current_status s
  ON s.case_version_id = cv.case_version_id
WHERE s.status = 'active';

-- v_dataset_contents: expand a dataset_version into its rows.
CREATE VIEW v_dataset_contents AS
SELECT
    dv.dataset_version_id,
    dv.dataset_name,
    (m.value->>'case_id')::text         AS case_id,
    (m.value->>'case_version_id')::text AS case_version_id,
    (m.ord)::int                        AS ordinal
FROM dataset_version dv,
     LATERAL jsonb_array_elements(dv.manifest) WITH ORDINALITY m(value, ord);
```

### 2.3 `tcm/manifest.py` — the content-addressed manifest

```python
"""Compute a deterministic dataset_version_id from a list of (case_id, case_version_id)
pairs. The hash covers the ordered list; reordering does NOT produce the same id."""
from __future__ import annotations
import hashlib
import json
from dataclasses import dataclass


@dataclass(frozen=True)
class CaseRef:
    case_id: str
    case_version_id: str


def manifest_bytes(cases: list[CaseRef]) -> bytes:
    rows = [{"case_id": c.case_id, "case_version_id": c.case_version_id} for c in cases]
    return json.dumps(rows, separators=(",", ":"), sort_keys=False).encode("utf-8")


def dataset_version_id(cases: list[CaseRef]) -> str:
    return "sha256:" + hashlib.sha256(manifest_bytes(cases)).hexdigest()


def content_sha256(input_obj: dict, expected_obj: dict, rubric_version: str) -> str:
    """Hash for a case_version. Normalise key order so semantically equal
    cases collapse to the same version id."""
    blob = json.dumps(
        {"input": input_obj, "expected": expected_obj, "rubric_version": rubric_version},
        separators=(",", ":"), sort_keys=True,
    ).encode("utf-8")
    return hashlib.sha256(blob).hexdigest()
```

### 2.4 `tcm/api.py` — the CRUD surface

```python
"""Test-case management API. One class, synchronous; backing store is a
psycopg connection. The write paths enforce the lifecycle rules: only
`active` cases can join a dataset_version with status='active'."""
from __future__ import annotations
import time
from dataclasses import dataclass
from typing import Any

from .manifest import CaseRef, content_sha256, dataset_version_id


ALLOWED_STATUS = {"draft", "candidate", "active", "retired"}
ALLOWED_TRANSITIONS = {
    "draft": {"candidate", "retired"},
    "candidate": {"active", "draft", "retired"},
    "active": {"retired"},               # once active, you cannot slide back
    "retired": set(),                    # retired is terminal for that version id
}
ALLOWED_CHANGE_KINDS = {"new", "fix", "drift_update", "retire"}


@dataclass(frozen=True)
class CaseVersionRow:
    case_id: str
    case_version_id: str
    input: dict[str, Any]
    expected: dict[str, Any]
    label_source: str
    rubric_version: str
    author: str
    change_kind: str
    rationale: str


class TestCaseManager:
    def __init__(self, conn) -> None:
        self._conn = conn

    # ---- case + case_version writes ----

    def upsert_case(self, case_id: str, surface: str, tags: list[str], created_by: str) -> None:
        self._conn.execute(
            """
            INSERT INTO "case" (case_id, surface, tags, created_ns, created_by)
            VALUES (%s, %s, %s, %s, %s)
            ON CONFLICT (case_id) DO NOTHING
            """,
            (case_id, surface, tags, time.time_ns(), created_by),
        )

    def write_case_version(self, v: CaseVersionRow, *, parent_version_id: str | None) -> str:
        if v.change_kind not in ALLOWED_CHANGE_KINDS:
            raise ValueError(f"change_kind must be one of {ALLOWED_CHANGE_KINDS}")
        if not v.rationale.strip():
            raise ValueError("rationale is required; the audit trail depends on it")
        sha = content_sha256(v.input, v.expected, v.rubric_version)
        case_version_id = "sha256:" + sha
        self._conn.execute(
            """
            INSERT INTO case_version (
                case_version_id, case_id, parent_version_id,
                input, expected, label_source, rubric_version,
                content_sha256, created_ns, author, change_kind, rationale
            )
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (case_version_id) DO NOTHING
            """,
            (
                case_version_id, v.case_id, parent_version_id,
                v.input, v.expected, v.label_source, v.rubric_version,
                sha, time.time_ns(), v.author, v.change_kind, v.rationale,
            ),
        )
        # First status is draft; promotion is a separate call.
        self._conn.execute(
            """
            INSERT INTO case_status (case_id, case_version_id, status, effective_ns, actor)
            VALUES (%s, %s, 'draft', %s, %s)
            ON CONFLICT DO NOTHING
            """,
            (v.case_id, case_version_id, time.time_ns(), v.author),
        )
        return case_version_id

    # ---- lifecycle transitions ----

    def transition(self, case_id: str, case_version_id: str, to_status: str, actor: str) -> None:
        if to_status not in ALLOWED_STATUS:
            raise ValueError(f"status must be one of {ALLOWED_STATUS}")
        cur = self._current_status(case_id, case_version_id)
        if cur is None:
            raise LookupError(f"no status rows for {case_id=} {case_version_id=}")
        if to_status not in ALLOWED_TRANSITIONS[cur]:
            raise ValueError(f"illegal transition {cur} -> {to_status}")
        self._conn.execute(
            """
            INSERT INTO case_status (case_id, case_version_id, status, effective_ns, actor)
            VALUES (%s, %s, %s, %s, %s)
            """,
            (case_id, case_version_id, to_status, time.time_ns(), actor),
        )

    def _current_status(self, case_id: str, case_version_id: str) -> str | None:
        row = self._conn.execute(
            """
            SELECT status FROM case_status
            WHERE case_id = %s AND case_version_id = %s
            ORDER BY effective_ns DESC LIMIT 1
            """,
            (case_id, case_version_id),
        ).fetchone()
        return row[0] if row else None

    # ---- dataset writes ----

    def cut_dataset(
        self, *, dataset_name: str, refs: list[CaseRef],
        created_by: str, change_notes: str, parent_version_id: str | None,
    ) -> str:
        """Compose a dataset_version from an ordered list of case refs.
        Every ref MUST currently be in status `active`; otherwise the write
        is rejected. The id is deterministic from the manifest."""
        active = self._active_case_set()
        missing = [r for r in refs if (r.case_id, r.case_version_id) not in active]
        if missing:
            raise ValueError(f"not-active cases in manifest: {missing[:5]}")
        dv_id = dataset_version_id(refs)
        self._conn.execute(
            """
            INSERT INTO dataset_version (
                dataset_version_id, dataset_name, manifest_sha256,
                manifest, row_count, parent_version_id,
                created_ns, created_by, change_notes
            )
            VALUES (%s, %s, %s, %s, %s, %s, %s, %s, %s)
            ON CONFLICT (dataset_version_id) DO NOTHING
            """,
            (
                dv_id, dataset_name, dv_id.removeprefix("sha256:"),
                [{"case_id": r.case_id, "case_version_id": r.case_version_id} for r in refs],
                len(refs), parent_version_id,
                time.time_ns(), created_by, change_notes,
            ),
        )
        return dv_id

    def _active_case_set(self) -> set[tuple[str, str]]:
        return {
            (row[0], row[1])
            for row in self._conn.execute(
                "SELECT case_id, case_version_id FROM v_active_cases"
            ).fetchall()
        }
```

### 2.5 `tcm/cli.py` — a thin CLI used by curators

```python
"""Operator commands. `tcm add` writes a case version (status draft), `tcm promote`
transitions it, `tcm cut` composes a dataset version. All destructive moves
require an explicit `--actor` and `--rationale`."""
from __future__ import annotations
import argparse
import json
import sys

from .api import CaseVersionRow, TestCaseManager
from .manifest import CaseRef


def _add(args, mgr: TestCaseManager) -> None:
    payload = json.load(args.file)
    mgr.upsert_case(args.case_id, args.surface, args.tag, args.actor)
    cv_id = mgr.write_case_version(
        CaseVersionRow(
            case_id=args.case_id,
            case_version_id="",   # filled by hash
            input=payload["input"],
            expected=payload["expected"],
            label_source=args.label_source,
            rubric_version=args.rubric_version,
            author=args.actor,
            change_kind=args.change_kind,
            rationale=args.rationale,
        ),
        parent_version_id=args.parent,
    )
    print(cv_id)


def _promote(args, mgr: TestCaseManager) -> None:
    mgr.transition(args.case_id, args.case_version_id, args.to, args.actor)


def _cut(args, mgr: TestCaseManager) -> None:
    refs = [CaseRef(**json.loads(line)) for line in args.manifest]
    dv = mgr.cut_dataset(
        dataset_name=args.name, refs=refs,
        created_by=args.actor, change_notes=args.notes,
        parent_version_id=args.parent,
    )
    print(dv)


def build_parser() -> argparse.ArgumentParser:
    p = argparse.ArgumentParser("tcm")
    sub = p.add_subparsers(required=True)

    a = sub.add_parser("add")
    a.add_argument("--case-id", required=True)
    a.add_argument("--surface", required=True)
    a.add_argument("--tag", action="append", default=[])
    a.add_argument("--label-source", choices=["human", "judge", "auto"], required=True)
    a.add_argument("--rubric-version", required=True)
    a.add_argument("--change-kind", choices=["new", "fix", "drift_update", "retire"], required=True)
    a.add_argument("--rationale", required=True)
    a.add_argument("--actor", required=True)
    a.add_argument("--parent", default=None)
    a.add_argument("--file", type=argparse.FileType("r"), default=sys.stdin)
    a.set_defaults(fn=_add)

    pr = sub.add_parser("promote")
    pr.add_argument("--case-id", required=True)
    pr.add_argument("--case-version-id", required=True)
    pr.add_argument("--to", choices=["candidate", "active", "retired", "draft"], required=True)
    pr.add_argument("--actor", required=True)
    pr.set_defaults(fn=_promote)

    c = sub.add_parser("cut")
    c.add_argument("--name", required=True)
    c.add_argument("--notes", required=True)
    c.add_argument("--actor", required=True)
    c.add_argument("--parent", default=None)
    c.add_argument("--manifest", type=argparse.FileType("r"), default=sys.stdin,
                   help="jsonl of {case_id, case_version_id} lines")
    c.set_defaults(fn=_cut)
    return p
```

### 2.6 `tests/test_lifecycle.py` — transitions enforced

```python
import pytest

from tcm.api import TestCaseManager, CaseVersionRow


def _seed(mgr: TestCaseManager) -> str:
    mgr.upsert_case("support.order_lookup.001", "support_agent", ["rag"], created_by="alice")
    return mgr.write_case_version(
        CaseVersionRow(
            case_id="support.order_lookup.001", case_version_id="",
            input={"message": "where's my order 12345?"},
            expected={"must_call_tool": "order_status"},
            label_source="human", rubric_version="rubric-v1",
            author="alice", change_kind="new", rationale="seed case",
        ),
        parent_version_id=None,
    )


def test_draft_cannot_jump_to_active(mgr):
    cv = _seed(mgr)
    with pytest.raises(ValueError, match="illegal transition draft -> active"):
        mgr.transition("support.order_lookup.001", cv, "active", actor="alice")


def test_candidate_then_active_is_allowed(mgr):
    cv = _seed(mgr)
    mgr.transition("support.order_lookup.001", cv, "candidate", actor="alice")
    mgr.transition("support.order_lookup.001", cv, "active", actor="bob")


def test_retired_is_terminal(mgr):
    cv = _seed(mgr)
    mgr.transition("support.order_lookup.001", cv, "candidate", actor="alice")
    mgr.transition("support.order_lookup.001", cv, "retired", actor="alice")
    with pytest.raises(ValueError, match="illegal transition retired -> active"):
        mgr.transition("support.order_lookup.001", cv, "active", actor="alice")
```

### 2.7 `tests/test_dataset_cut.py` — only active cases can enter a dataset

```python
import pytest
from tcm.api import TestCaseManager, CaseVersionRow
from tcm.manifest import CaseRef


def test_cut_rejects_draft(mgr):
    mgr.upsert_case("support.a", "support_agent", [], created_by="alice")
    cv = mgr.write_case_version(
        CaseVersionRow(
            case_id="support.a", case_version_id="",
            input={"x": 1}, expected={"y": 2},
            label_source="human", rubric_version="rubric-v1",
            author="alice", change_kind="new", rationale="seed",
        ),
        parent_version_id=None,
    )
    # cv is draft by default
    with pytest.raises(ValueError, match="not-active"):
        mgr.cut_dataset(
            dataset_name="support.regression",
            refs=[CaseRef(case_id="support.a", case_version_id=cv)],
            created_by="alice", change_notes="initial", parent_version_id=None,
        )


def test_cut_is_deterministic(mgr):
    # Two cuts with the same (ordered) manifest and schema produce the same id.
    mgr.upsert_case("support.a", "support_agent", [], created_by="alice")
    cv = mgr.write_case_version(
        CaseVersionRow(
            case_id="support.a", case_version_id="",
            input={"x": 1}, expected={"y": 2},
            label_source="human", rubric_version="rubric-v1",
            author="alice", change_kind="new", rationale="seed",
        ),
        parent_version_id=None,
    )
    mgr.transition("support.a", cv, "candidate", actor="alice")
    mgr.transition("support.a", cv, "active", actor="alice")
    refs = [CaseRef(case_id="support.a", case_version_id=cv)]
    dv1 = mgr.cut_dataset(dataset_name="d", refs=refs, created_by="alice",
                          change_notes="one", parent_version_id=None)
    dv2 = mgr.cut_dataset(dataset_name="d", refs=refs, created_by="alice",
                          change_notes="one-again", parent_version_id=None)
    assert dv1 == dv2
```

### 2.8 `docs/review_workflow.md` — the governance loop

```markdown
# Review workflow for test-case changes

Every `fix` or `drift_update` change kind must land through a reviewed PR
in the test-case repo. The owner is whoever appears in CODEOWNERS for the
case's surface. The PR body MUST state:

1. The `change_kind` (fix | drift_update | retire).
2. The `rationale` (why this change exists).
3. The failing or drifting evidence (link to a verdict, a user report,
   an incident).
4. The expected shift in CI gate metrics for the affected dataset.

A case version lands in `draft`. Promoting to `candidate` runs the case in
shadow (scored but not blocking) for the next regression cycle. Promoting
to `active` requires:

- Two reviewers.
- Green shadow-run verdict distribution (no > 1σ drop vs. the prior version).
- The rubric version matches the current platform rubric.

Retirement does not delete the case; it marks it unavailable for new
`dataset_version` writes. Historical verdicts remain queryable.
```

### 2.9 Worked walk-through — fixing a drifted gold answer

Flow:

1. A user report says "the support bot now answers 'order 12345' with the right ticket id, but our regression says it's wrong." Root cause: the gold answer has an outdated ticket id format.
2. Curator opens a PR editing the expected value for `support.order_lookup.001`.
3. `tcm add --change-kind=drift_update --rationale="ticket id format changed in 2026-11-01 release; gold was wrong" ...` writes a new `case_version` row with a new `case_version_id` (new content SHA), keyed back to the same `case_id`. Parent is the old version.
4. Status is `draft`; promotion to `candidate` runs the case in shadow on the next scheduled regression (exercise-03). Shadow verdicts show pass.
5. Promote to `active` after two reviews. Old version is auto-transitioned to `retired` by the transition hook.
6. The next `tcm cut` for the `support.regression` dataset picks up the new version; the dataset's own id changes (content hash of the ordered manifest). CI (exercise-05) now runs the new golden answer.
7. Historical `eval_result` rows still point at the old `dataset_version_id`; nothing is overwritten. The "why did this verdict change?" view (exercise-01, §2.4) joins back and shows the two dataset ids side-by-side.

## 3. Validation steps

1. **Load the DDL**: `psql -f schemas/case.sql -f schemas/views.sql`. No errors.
2. **Add and promote**: `tcm add --case-id=support.a --change-kind=new --rationale="seed" ... < sample.json` emits a `sha256:...`. `tcm promote --to=candidate` and `--to=active` succeed in order. Skipping candidate fails with the expected transition error.
3. **Cut a dataset**: pipe a manifest of two active cases to `tcm cut --name=support.regression --notes=initial`. Expect a `sha256:` id. Rerun with the same manifest; expect the same id (determinism). Change ordering; expect a different id.
4. **Reject draft inclusion**: write a new case version (draft) and try to include it in a cut. Expect `not-active` error.
5. **Audit queries**:
    - `SELECT change_kind, COUNT(*) FROM case_version GROUP BY 1` — expect `new`, `fix`, `drift_update`, `retire` present when the suite has run.
    - `SELECT case_id, COUNT(*) AS versions FROM case_version GROUP BY 1 ORDER BY versions DESC LIMIT 10` — the most-edited cases surface drift hotspots.
6. **Lineage join**: `SELECT * FROM v_dataset_contents WHERE dataset_version_id = '<id>'` returns the ordered list; joining to `case_version` yields the input/expected pairs.
7. **Retirement path**: `tcm promote --to=retired` on an `active` version succeeds. A subsequent `tcm cut` that references that version id fails with `not-active`.
8. **Run the pytest suite**: `pytest -q tests/`. All lifecycle + cut-determinism tests pass.

## 4. Rubric / review checklist

| #   | Criterion                                                                                               | Pass                                                                             | Fail                                                                     |
| --- | ------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------- | ------------------------------------------------------------------------ |
| R1  | Case identity is stable (`case_id`) while every version is immutable (`case_version_id`).                | DDL splits `case` and `case_version`; edits produce new `case_version_id`.       | Edits mutate a single row; no versioning.                                 |
| R2  | `case_version_id` is a content hash of `(input, expected, rubric_version)`.                              | `content_sha256` test proves two writers converge.                               | IDs are UUIDs or sequence numbers; equal content yields different ids.   |
| R3  | Dataset versions are content-addressed manifests; same ordered manifest → same id.                       | `test_cut_is_deterministic` passes.                                              | Dataset ids drift on re-cut; manifests not hashed.                       |
| R4  | Lifecycle is explicit and transitions are enforced: `draft → candidate → active → retired`.              | `ALLOWED_TRANSITIONS` enforced by the API; lifecycle tests pass.                 | Any status can jump to any status; active content can be edited in place. |
| R5  | Only `active` cases can participate in a new `dataset_version`.                                          | `cut_dataset` rejects drafts; test proves rejection.                              | Dataset cut accepts any version; drafts leak into CI.                     |
| R6  | Every write records `author`, `change_kind`, `rationale`.                                                | NOT NULL in `case_version`; the API rejects empty rationale.                      | Edits land without an author or reason.                                   |
| R7  | Label provenance is first-class: `label_source`, `rubric_version`, optional `annotator_id_hash`.          | Columns present on `case_version`; tests cover them.                              | Labels stored as raw strings, no source or rubric.                        |
| R8  | Retired cases stay queryable; `retired` is a terminal transition that does not delete history.           | DDL keeps retired rows; promotion to `active` from `retired` is rejected.         | Retirement deletes rows; history breaks.                                  |
| R9  | Dataset-contents view (`v_dataset_contents`) returns the ordered manifest for a given dataset version.   | View returns rows with `ordinal`.                                                 | Consumers must parse the JSON manifest by hand.                           |
| R10 | Governance docs describe the review workflow and the CODEOWNERS expectation.                             | `docs/review_workflow.md` present and explicit.                                   | No review workflow; drift fixes can land solo.                            |

A passing submission is 8+ of 10 with R1, R2, R4, R5, R6 all hard-required.

## 5. Common mistakes

- **Mutating golden answers in place.** The most common cause of unexplained CI regressions. Every edit is a new version; the old one remains linked to past verdicts.
- **UUIDs for case versions.** Non-content-addressed ids mean two writers produce different ids for identical content, which defeats deduplication and makes CI flaky.
- **No parent pointer.** `parent_version_id` is cheap and makes the "what did this used to say?" query a single row read. Skipping it forces log-digging.
- **No rationale on fixes.** Six months later nobody remembers why the gold answer changed. The audit trail is cheaper than the postmortem.
- **One status column on the case table.** A single mutable status column loses history; a status log preserves it and is still cheap to read with a view.
- **No rubric version.** A label written against rubric v1 and compared under rubric v2 is a silent bug. Store the rubric id on the label.
- **Labels without a source.** `label_source` separates human-labeled from judge-labeled from auto-labeled rows; they drift at different rates and deserve different trust weights in downstream analyses.
- **No manifest ordering.** If two writers produce the same case set in different orders and get the same id, you lose the ability to run tests in a declared order (useful for first-N smoke suites).
- **Retire-by-delete.** Removing retired rows breaks lineage joins from old `eval_result` rows. Retire by status, keep the row.
- **Mixing drafts and active in one dataset.** A dataset cut with mixed-status rows is impossible to interpret under a governance review. Only active rows enter a published cut.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `GOVERN 2.1` (accountability), `MAP 1.1` (dataset lineage), `MEASURE 2.3` (metric coverage) frame the lifecycle and provenance requirements.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning) motivates content addressing and the two-reviewer rule.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.M0015` (prompt hardening) frames why the gold answer, its rubric, and its author belong together in the audit trail.
- JSON Schema draft 2020-12, <https://json-schema.org/draft/2020-12/schema>.
- Local exercise reader: `lessons/mod-110-eval-data-platform-slice/exercises/exercise-02-test-case-management-versioning.md`.
- Related solutions: exercise-01 consumes `dataset_version_id` on `eval_result`; exercise-03 picks `active` dataset versions for its runs; exercise-05 blocks CI when the active dataset's verdict distribution regresses.
