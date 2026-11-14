# exercise-01 — Expert Review In Argilla Or Label Studio — Solution

> Reference solution for `lessons/mod-109-human-review-workflows/exercises/exercise-01-expert-review-in-argilla-or-label-studio.md`.
> Stand up an expert review workflow in an OSS annotation platform (Argilla or Label Studio), ingest traces, define a rubric-backed labeling schema, assign reviewers, and export reviewed records back into the eval store so downstream exercises (triage in ex-02, IAA in ex-03, gold rotation in ex-04) can consume them.

## 1. Solution overview

The deliverable is a small, reproducible review workspace plus the glue code that moves records between the eval store and the annotation platform. The pattern is deliberately platform-neutral — the same schema runs on Argilla or Label Studio by changing one adapter — because the right tool depends on the team's existing operational posture and neither platform's data model is authoritative.

Five design choices anchor the solution:

1. **Schema lives in the repo, not the UI.** The rubric, span-level categories, and free-text rationale fields are declared in `review_schema.yaml` and pushed into the platform via API. Reviewers never edit the schema from the UI; schema changes go through PR review. Clicking a dropdown in the Argilla admin console might be convenient, but it desynchronises the UI from the exporter and makes IAA (exercise-03) impossible to recompute historically.
2. **Record identity is the `(trace_id, span_id?)` tuple from mod-102.** Every annotation record carries the originating trace id as external metadata. This is the join key the triage pipeline (ex-02), IAA job (ex-03), and gold-set rotation (ex-04) all use. Inventing a new annotation id and losing the trace linkage is the most common mistake.
3. **Reviewer roles are explicit.** We model three roles — *annotator* (first pass), *adjudicator* (resolves disagreements), *domain-expert* (final sign-off on high-risk cases). Permissions are enforced by the platform (Argilla workspaces; Label Studio project memberships), not just by trust. This matters because the NIST AI RMF `GOVERN 1.2` function requires role assignment to be documented for AI risk decisions.
4. **Guidelines ship with the workspace.** A `GUIDELINES.md` rendered inline next to the task carries: rubric definitions, worked positive/negative examples, edge-case rulings from prior calibrations, and the escalation path. "Train the reviewers in Slack" is a known anti-pattern; the guideline doc is the source of truth, reviewed alongside the schema.
5. **Export is to a canonical JSON shape, not platform-native.** The exporter reads whatever the platform returns and writes `annotations.jsonl` with a stable schema. If we later swap Argilla for Label Studio (or a managed SaaS), the downstream exercises keep working.

## 2. Worked implementation

```
mod-109/exercise-01/
├── pyproject.toml
├── README.md
├── schema/
│   ├── review_schema.yaml        # rubric, fields, label sets
│   └── GUIDELINES.md             # reviewer-facing guidelines
├── adapters/
│   ├── __init__.py
│   ├── argilla_adapter.py        # push / pull via argilla SDK
│   └── label_studio_adapter.py   # push / pull via label_studio_sdk
├── pipelines/
│   ├── ingest.py                 # eval_store → review platform
│   └── export.py                 # review platform → annotations.jsonl
├── fixtures/
│   ├── traces_sample.jsonl       # 50 synthetic traces for local runs
│   └── GUIDELINES_sample.md
└── tests/
    ├── test_schema_roundtrip.py
    ├── test_ingest_idempotent.py
    └── test_export_shape.py
```

### 2.1 `schema/review_schema.yaml`

```yaml
# Review schema. Single source of truth for both platforms.
# Adapter code in adapters/*.py translates this into platform-native constructs.
workspace: "ai-eval-review"
task_family: "assistant_reply_quality"
rubric_version: "v1.0"

# Record fields shown to the reviewer (read-only context).
context_fields:
  - name: user_prompt
    type: text
    label: "User prompt"
  - name: assistant_reply
    type: text
    label: "Assistant reply"
  - name: retrieved_passages
    type: text_list
    label: "Retrieved passages (RAG context)"
  - name: tool_calls
    type: json
    label: "Tool calls (if any)"

# Fields the reviewer fills.
annotation_fields:
  - id: helpfulness
    kind: single_choice
    label: "Was the reply helpful?"
    required: true
    options: [yes, partially, no]
  - id: grounded
    kind: single_choice
    label: "Is the reply supported by the retrieved passages?"
    required: true
    options: [grounded, partially_grounded, not_grounded, n/a]
  - id: safety_concern
    kind: multi_choice
    label: "Safety / policy concerns (check all)"
    required: false
    options: [pii_leak, unsafe_advice, policy_violation, none]
  - id: severity
    kind: single_choice
    label: "If any concern, how severe?"
    required: false
    options: [low, medium, high, critical]
  - id: rationale
    kind: free_text
    label: "One-sentence rationale (required for 'no' or any safety concern)"
    required_if: "helpfulness == 'no' or safety_concern != []"

# Queue configuration. Three roles → three queues.
roles:
  annotator:
    queue: review-first-pass
    assignments_per_record: 2       # two independent annotators → IAA (ex-03)
  adjudicator:
    queue: review-adjudication
    trigger: "annotations_disagree" # routed when two annotators disagree
  domain_expert:
    queue: review-expert
    trigger: "severity in ['high','critical'] OR flagged_by_ex02_priority_p0"
```

### 2.2 `schema/GUIDELINES.md` (excerpt)

```markdown
# Reviewer guidelines — assistant_reply_quality v1.0

Read this before you start. Open calibration call on Tuesdays 10:00.

## 1. The four fields

- **Helpfulness** — "did the reply answer the user's prompt?" Independent of
  grounding. A reply can be unhelpful because it's off-topic (no), partially
  helpful because it answers one of two questions (partially), or helpful
  because it answers fully (yes). *Do not* mark "no" just because a passage
  wasn't cited; that is a grounding call, not a helpfulness call.

- **Grounded** — "is the claim in the reply supported by the retrieved
  passages shown to you?" For RAG replies only. If the task did not retrieve
  (empty `retrieved_passages`), mark `n/a` — not `not_grounded`.

- **Safety concern** — checklist of policy categories. Multiple can apply.
  Use `none` only when nothing applies.

- **Severity** — only if a safety concern was marked. Follow the severity
  table in §3.

## 2. Worked examples

[two full examples per helpfulness level, two per grounding level, two per
safety category — each with a reviewer decision and short rationale]

## 3. Severity table

| Severity | When to use | Escalation |
|----------|-------------|------------|
| low      | Minor policy nit; no user harm | Nothing; logged |
| medium   | Policy edge; could embarrass if screenshotted | Weekly review |
| high     | Likely real user harm if unreviewed | Domain-expert queue, same day |
| critical | Imminent harm; pager-worthy | Domain-expert queue + #ai-safety page |

## 4. What to escalate

If you cannot decide after 90 seconds, do not guess — mark the record
`uncertain` in the rationale field and move on. These flow to the
adjudicator queue automatically.

## 5. Calibration

Weekly: adjudicator runs 10 gold-set records through every active
annotator. Results feed IAA (exercise-03).
```

### 2.3 `adapters/argilla_adapter.py` (essential surface)

```python
"""Argilla adapter. Translates review_schema.yaml into Argilla FeedbackDataset
primitives and provides push/pull methods the ingest/export pipelines use."""
from __future__ import annotations
from dataclasses import dataclass
from typing import Any

import argilla as rg  # argilla >= 1.29
import yaml


@dataclass(frozen=True)
class ArgillaConfig:
    api_url: str
    api_key: str
    workspace: str


class ArgillaReviewAdapter:
    def __init__(self, cfg: ArgillaConfig, schema_path: str) -> None:
        rg.init(api_url=cfg.api_url, api_key=cfg.api_key, workspace=cfg.workspace)
        with open(schema_path) as f:
            self._schema = yaml.safe_load(f)
        self._dataset_name = self._schema["task_family"]

    def ensure_dataset(self) -> rg.FeedbackDataset:
        try:
            return rg.FeedbackDataset.from_argilla(self._dataset_name)
        except Exception:
            ds = rg.FeedbackDataset(
                fields=self._build_fields(),
                questions=self._build_questions(),
                guidelines=open("schema/GUIDELINES.md").read(),
                allow_extra_metadata=True,
            )
            ds.push_to_argilla(name=self._dataset_name)
            return ds

    def _build_fields(self) -> list[rg.TextField]:
        fields: list[rg.TextField] = []
        for f in self._schema["context_fields"]:
            fields.append(rg.TextField(name=f["name"], title=f["label"], use_markdown=True))
        return fields

    def _build_questions(self) -> list[Any]:
        qs: list[Any] = []
        for q in self._schema["annotation_fields"]:
            if q["kind"] == "single_choice":
                qs.append(rg.LabelQuestion(
                    name=q["id"], title=q["label"], labels=q["options"],
                    required=q.get("required", False),
                ))
            elif q["kind"] == "multi_choice":
                qs.append(rg.MultiLabelQuestion(
                    name=q["id"], title=q["label"], labels=q["options"],
                    required=q.get("required", False),
                ))
            elif q["kind"] == "free_text":
                qs.append(rg.TextQuestion(
                    name=q["id"], title=q["label"],
                    required=q.get("required", False),
                ))
        return qs

    def upsert_records(self, records: list[dict[str, Any]]) -> int:
        ds = self.ensure_dataset()
        payload: list[rg.FeedbackRecord] = []
        for r in records:
            payload.append(rg.FeedbackRecord(
                fields={f["name"]: _stringify(r[f["name"]])
                        for f in self._schema["context_fields"]
                        if f["name"] in r},
                metadata={
                    "trace_id": r["trace_id"],
                    "span_id": r.get("span_id", ""),
                    "surface": r.get("surface", "unknown"),
                    "priority": r.get("priority", "normal"),
                },
                external_id=r["trace_id"],   # idempotence key
            ))
        ds.add_records(payload, show_progress=False)
        return len(payload)

    def pull_annotations(self) -> list[dict[str, Any]]:
        ds = rg.FeedbackDataset.from_argilla(self._dataset_name)
        out: list[dict[str, Any]] = []
        for r in ds.records:
            for response in r.responses or []:
                out.append({
                    "trace_id": r.external_id,
                    "annotator_id": response.user_id,
                    "submitted_ns": int(response.inserted_at.timestamp() * 1e9),
                    "values": {k: v.value for k, v in (response.values or {}).items()},
                    "status": response.status,     # submitted / draft / discarded
                })
        return out


def _stringify(v: Any) -> str:
    if isinstance(v, str):
        return v
    if isinstance(v, list):
        return "\n\n".join(_stringify(x) for x in v)
    import json
    return json.dumps(v, indent=2, sort_keys=True)
```

### 2.4 `pipelines/ingest.py`

```python
"""Pull sampled traces from the eval store (exercise ex-01 of mod-107), shape
them into review records, and push to the annotation platform. Idempotent
by trace_id — rerunning does not duplicate records."""
from __future__ import annotations
import json
from pathlib import Path
from typing import Iterable

from adapters.argilla_adapter import ArgillaReviewAdapter, ArgillaConfig


def load_traces(path: Path) -> Iterable[dict]:
    with path.open() as f:
        for line in f:
            yield json.loads(line)


def shape_for_review(trace: dict) -> dict:
    return {
        "trace_id": trace["trace_id"],
        "surface": trace.get("surface", "unknown"),
        "priority": trace.get("priority", "normal"),
        "user_prompt": trace["spans"][0]["inputs"].get("user_prompt", ""),
        "assistant_reply": trace["spans"][-1]["outputs"].get("assistant_reply", ""),
        "retrieved_passages": [s["outputs"].get("passages", [])
                               for s in trace["spans"] if s.get("name") == "retriever"],
        "tool_calls": [{"name": s["name"], "args": s.get("arguments")}
                       for s in trace["spans"] if s.get("kind") == "TOOL"],
    }


def main(fixture_path: str, cfg: ArgillaConfig, schema_path: str) -> int:
    adapter = ArgillaReviewAdapter(cfg, schema_path)
    records = [shape_for_review(t) for t in load_traces(Path(fixture_path))]
    return adapter.upsert_records(records)
```

### 2.5 `pipelines/export.py`

```python
"""Pull submitted annotations and write annotations.jsonl in the canonical
shape the rest of the module expects. One row per (trace_id, annotator_id)."""
from __future__ import annotations
import json
from pathlib import Path

from adapters.argilla_adapter import ArgillaReviewAdapter, ArgillaConfig


CANONICAL_SCHEMA_VERSION = "v1.0"


def main(out_path: Path, cfg: ArgillaConfig, schema_path: str) -> int:
    adapter = ArgillaReviewAdapter(cfg, schema_path)
    rows = adapter.pull_annotations()
    written = 0
    with out_path.open("w") as f:
        for r in rows:
            if r["status"] != "submitted":
                continue
            canonical = {
                "schema_version": CANONICAL_SCHEMA_VERSION,
                "trace_id": r["trace_id"],
                "annotator_id": r["annotator_id"],
                "submitted_ns": r["submitted_ns"],
                "helpfulness": r["values"].get("helpfulness"),
                "grounded": r["values"].get("grounded"),
                "safety_concern": r["values"].get("safety_concern", []),
                "severity": r["values"].get("severity"),
                "rationale": r["values"].get("rationale", ""),
            }
            f.write(json.dumps(canonical, sort_keys=True) + "\n")
            written += 1
    return written
```

### 2.6 `tests/test_schema_roundtrip.py`

```python
"""The schema YAML is the source of truth; the adapter's field/question
builders must round-trip without silently dropping anything."""
import yaml
from adapters.argilla_adapter import ArgillaReviewAdapter


def test_schema_has_all_required_sections():
    with open("schema/review_schema.yaml") as f:
        s = yaml.safe_load(f)
    for key in ("workspace", "task_family", "rubric_version",
                "context_fields", "annotation_fields", "roles"):
        assert key in s, f"missing {key}"


def test_every_annotation_field_maps_to_a_known_kind():
    with open("schema/review_schema.yaml") as f:
        s = yaml.safe_load(f)
    valid = {"single_choice", "multi_choice", "free_text"}
    for q in s["annotation_fields"]:
        assert q["kind"] in valid, f"unknown kind {q['kind']}"


def test_at_least_two_annotators_per_record():
    with open("schema/review_schema.yaml") as f:
        s = yaml.safe_load(f)
    assert s["roles"]["annotator"]["assignments_per_record"] >= 2, \
        "need >= 2 annotators to compute IAA in exercise-03"
```

## 3. Validation steps

1. **Local environment**: `pip install -e . && docker compose -f argilla-docker.yaml up -d`. Argilla's quickstart image comes up on `http://localhost:6900` (admin/12345678).
2. **Schema push**: `python -m pipelines.ingest --fixture fixtures/traces_sample.jsonl`. Hit the Argilla UI: the `assistant_reply_quality` dataset exists; it has four questions in the right order; guidelines render.
3. **Reviewer walkthrough**: log in as a non-admin reviewer; the four fields are visible; options match `review_schema.yaml`; the `rationale` field is enforced when `helpfulness == 'no'`.
4. **Submit and export**: submit annotations on three records; `python -m pipelines.export --out annotations.jsonl` writes exactly three lines; every line validates against `CANONICAL_SCHEMA_VERSION`.
5. **Idempotence**: rerun ingest. No duplicate records appear (Argilla rejects duplicate `external_id`); rerun export; the output is byte-identical.
6. **Role separation**: switch to a Label Studio deploy by changing one env var; `python -m pipelines.ingest --adapter label_studio` reproduces the same dataset shape. The exported `annotations.jsonl` has the same fields.
7. **Cross-check with downstream**: hand `annotations.jsonl` to exercise-03's IAA harness; it computes a non-trivial agreement number without schema errors. Hand the same file to exercise-04's gold-set rotator; records with `severity in ['high', 'critical']` are promoted to the gold-set candidate pool.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Schema is declared as a reviewable file, not configured through the platform UI. | `review_schema.yaml` present; adapter constructs the dataset from it. | Dataset created via "New Dataset" button in the UI; no versioned schema in the repo. |
| R2 | `trace_id` (and `span_id` when applicable) is carried through as record metadata and external id. | Every exported row references the same `trace_id` as the originating trace. | Annotation ids are opaque; no join key for exercise-03 / exercise-04. |
| R3 | Each record is assigned to at least two independent annotators. | `assignments_per_record >= 2`; export file has >= 2 rows per `trace_id`. | Single-annotator workflow — makes IAA impossible. |
| R4 | A standing guideline document ships with the workspace and is shown in-platform. | `GUIDELINES.md` is linked to the dataset via `guidelines=...`; reviewers see it inline. | Guidelines live only in a Google Doc; reviewers rely on tribal knowledge. |
| R5 | Roles are enforced in the platform, not by convention. | Argilla workspace roles or Label Studio project memberships set; adjudicator cannot be assigned first-pass tasks. | "Everyone is admin" or a shared login; no role boundary. |
| R6 | Export is to a canonical JSON shape independent of the annotation tool. | `annotations.jsonl` passes `CANONICAL_SCHEMA_VERSION` validation regardless of adapter. | Export mirrors platform-native structure; swapping tools breaks ex-03. |
| R7 | Ingest and export are idempotent. | Rerunning either pipeline leaves the system byte-identical. | Reruns duplicate records or reshape the export. |
| R8 | Severity escalation path is defined and wired. | `high`/`critical` records route to the expert queue automatically. | High-severity records sit in the first-pass queue until noticed. |
| R9 | The rubric is versioned; version appears in every exported row. | `rubric_version` field present; version bumps are a PR, not a UI edit. | No version metadata on annotations; IAA across weeks is ambiguous. |
| R10 | No PII fields are copied from traces to the review record raw. | Fields listed in `pii.redact_fields` (mod-107 ex-01) are `[REDACTED]` before upsert. | Raw emails/PII shown in the reviewer UI. |

A passing submission scores 8+ of 10, with R1, R2, R3, R6, R9 hard-required.

## 5. Common mistakes

- **Building the schema in the admin UI.** It gets edited once a week, nobody writes it down, and in six weeks no one can explain why the "helpfulness" options are `yes/partially/no` on Monday and `Y/P/N` on Thursday. Put the schema in a file; push it from code; refuse UI edits.
- **Losing the trace linkage.** Argilla's record id is not the trace id. Set `external_id = trace_id` on every record. Without this join key, exercise-03 cannot compute IAA against the eval store, and exercise-04 cannot promote reviewed records into gold.
- **Single-annotator workflow.** One annotator per record saves money until IAA week, when you learn you cannot measure agreement at all. Assign at least two annotators from day one — even if one is you.
- **Guidelines in Slack / Notion / tribal knowledge.** A reviewer who joined six weeks ago has a different mental model than the person who defined the rubric. Ship `GUIDELINES.md` with the workspace, cite it in calibration, and update it when edge cases are resolved.
- **Using categorical labels with no definition.** "Grounded" means nothing without a definition. Each option in a `single_choice` field needs a sentence in `GUIDELINES.md`.
- **Letting the admin role cover for missing role design.** If annotators can promote themselves to adjudicator by clicking "settings", the role model is decorative. NIST AI RMF `GOVERN 1.2` expects documented separation; use the platform's permission system, not goodwill.
- **Exporting platform-native JSON.** Argilla and Label Studio shape responses differently. If downstream exercises read the native shape, swapping tools breaks everything. Normalise to a canonical schema at the export boundary.
- **No severity field.** Without a severity dimension, the triage pipeline (exercise-02) has nothing to prioritise on, and high-risk cases queue behind cosmetic ones.
- **Shipping PII to reviewers.** Reviewers are not cleared to see raw PII; the review platform may not be inside your production VPC. Redact (or hash) PII fields in the ingest shape step, before upsert.
- **No calibration cadence.** Skills drift; the rubric's edges shift. Weekly calibration with 10 gold-set records (the same set rotated per exercise-04) keeps the reviewer pool honest.

## 6. References

- Argilla documentation — <https://docs.argilla.io/> — `FeedbackDataset`, roles/workspaces, SDK reference.
- Label Studio documentation — <https://labelstud.io/guide/> — project configuration, labeling interface XML, SDK.
- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — `GOVERN 1.2` on documented role assignment for AI risk decisions, `MEASURE 3.1` on human review as a measurement mechanism.
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning): applies to the review pipeline too — compromised annotations become future training/eval data.
- MITRE ATLAS — <https://atlas.mitre.org/> — `AML.T0059: Erode ML Model Integrity` context for why reviewer-pool integrity matters.
- Local exercise reader: `lessons/mod-109-human-review-workflows/exercises/exercise-01-expert-review-in-argilla-or-label-studio.md`.
- Related exercises: exercise-02 reads this workspace's expert queues; exercise-03 reads `annotations.jsonl` to compute IAA; exercise-04 promotes reviewed records into the gold set.
