# exercise-01 — Trace and Eval-Result Schema with Lineage — Solution

> Reference solution for `lessons/mod-110-eval-data-platform-slice/exercises/exercise-01-trace-and-eval-result-schema-with-lineage.md`.
> This is a **design** exercise. The deliverable is a canonical data model for the platform's two foundational tables — `trace` and `eval_result` — plus an explicit lineage graph that lets any verdict be traced back to the exact app version, dataset snapshot, prompt template, and evaluator version that produced it. Downstream exercises (test-case versioning in exercise-02, multi-runner orchestration in exercise-03, SLOs in exercise-04, CI gating in exercise-05) all read against this contract.

## 1. Solution overview

The platform stores two primary fact tables and five lineage dimension tables. The design centers on five load-bearing choices:

1. **One `trace` row per interaction, one `eval_result` row per `(trace, evaluator, evaluator_version)`.** Verdicts are an append-mostly projection of traces; they never overwrite a trace and they never silently version-bump.
2. **Lineage lives in the keys, not in a "metadata" blob.** Every `eval_result` carries foreign keys to `dataset_version`, `prompt_version`, `app_version`, `evaluator_version`, and `run_id`. If any one is missing, the row is rejected at write time, not at analysis time.
3. **Content IDs are hashes.** Prompts, datasets, and app configs are content-addressed. Two `prompt_version` rows with the same hash collapse to one; two rows with different hashes are different versions, even if the human called them both "v2". This is the [NIST AI RMF](https://www.nist.gov/itl/ai-risk-management-framework) `GOVERN 1.6` traceability requirement expressed as a primary key.
4. **PII boundary is in the schema, not in the reader.** `trace` carries a hashed `user.id` and a `redaction_policy_version`; the raw PII never lands in the eval-plane store. OWASP ML Top 10 `ML02` (data poisoning) and MITRE ATLAS `AML.T0024` (LLM exfiltration) are both policed at the write boundary.
5. **Schema version is a first-class column on every row.** A `schema.version` bump is semver: additive attributes are minor bumps and do not change existing readers; renames or type changes are major bumps and require a reader migration plan.

The two fact tables and the five lineage tables, in one picture:

```
                     ┌────────────────────┐
                     │   dataset_version  │◄─────┐
                     │ (content-addressed)│      │
                     └────────────────────┘      │
                     ┌────────────────────┐      │
                     │   prompt_version   │◄─────┤
                     └────────────────────┘      │
                     ┌────────────────────┐      │  (FKs on every
                     │     app_version    │◄─────┤   eval_result)
                     └────────────────────┘      │
                     ┌────────────────────┐      │
                     │ evaluator_version  │◄─────┤
                     └────────────────────┘      │
                     ┌────────────────────┐      │
                     │        run         │◄─────┘
                     └────────────────────┘
                                ▲
             ┌──────────────────┴──────────────────┐
             │                                     │
  ┌──────────┴─────────┐                 ┌─────────┴──────────┐
  │       trace        │◄────────────────│     eval_result    │
  │ (one per request)  │  trace_id (FK)  │(one per verdict)   │
  └────────────────────┘                 └────────────────────┘
```

## 2. Worked implementation

### 2.1 `schemas/trace.sql` — the `trace` fact table

```sql
-- One row per end-user interaction (one span tree). This table is the single
-- source of truth; everything else in the eval plane references a trace by id.
-- Design rules (enforced):
--   * `trace_id` is the only visible identifier. It is a UUIDv7 so writes
--     stay append-friendly on time-ordered storage.
--   * PII fields never land here raw. `user_id_hash` is HMAC-SHA256 with a
--     per-tenant salt; `session_id_hash` the same. Raw IDs stay in the app
--     plane. See `redaction_policy_version` for the rule set in force.
--   * `app_version` + `prompt_version` are REQUIRED. If the writer cannot
--     supply them, the row is rejected at ingestion. There is no silent
--     "unknown" bucket.
--   * `schema_version` is semver (see §2.7).

CREATE TABLE trace (
    trace_id                UUID         PRIMARY KEY,
    tenant_id               TEXT         NOT NULL,
    tenant_class            TEXT         NOT NULL,         -- paid | free | internal
    surface                 TEXT         NOT NULL,         -- e.g. "support_agent"
    app_version_id          TEXT         NOT NULL REFERENCES app_version(app_version_id),
    prompt_version_id       TEXT         NOT NULL REFERENCES prompt_version(prompt_version_id),

    -- Identity (never raw).
    user_id_hash            CHAR(16)     NOT NULL,         -- first 16 hex of HMAC-SHA256
    session_id_hash         CHAR(16)     NOT NULL,
    redaction_policy_version TEXT        NOT NULL,         -- which PII policy produced those hashes

    -- Lifecycle.
    started_ns              BIGINT       NOT NULL,         -- otel start time
    duration_ns             BIGINT       NOT NULL,
    status                  TEXT         NOT NULL,         -- ok | error | timeout
    tool_shape              TEXT         NOT NULL,         -- chat_only | tool_call | rag | multi_tool

    -- Cost / usage rollup (so eval_result does not need to re-sum spans).
    tokens_prompt           INT          NOT NULL,
    tokens_completion       INT          NOT NULL,
    cost_usd                NUMERIC(10,6) NOT NULL,

    -- Payload pointer. The full span tree lives in object storage (parquet
    -- per hour). Payload contains redacted content only.
    payload_uri             TEXT         NOT NULL,
    payload_sha256          CHAR(64)     NOT NULL,

    schema_version          TEXT         NOT NULL,         -- e.g. "1.0.0"
    ingested_ns             BIGINT       NOT NULL
);

CREATE INDEX trace_by_time              ON trace (started_ns);
CREATE INDEX trace_by_tenant_surface    ON trace (tenant_id, surface, started_ns);
CREATE INDEX trace_by_app_version       ON trace (app_version_id, started_ns);
```

### 2.2 `schemas/eval_result.sql` — the verdict fact table

```sql
-- One row per (trace, evaluator, evaluator_version). Primary key includes the
-- evaluator version on purpose: bumping a judge does NOT overwrite history.
-- Design rules:
--   * Every foreign key must resolve. There is no NULL app_version_id or
--     dataset_version_id; if we cannot attribute the verdict, we do not
--     write it.
--   * `run_id` is the batch / stream run that produced the verdict. The
--     same evaluator can run under CI (exercise-05), online (mod-107 ex-01),
--     or scheduled regression (exercise-03) — each gets its own run_id but
--     the same `eval_result` shape.
--   * `status` is a first-class enum, including `timeout` and `over_budget`.
--     Dropping the row corrupts drift denominators; always write a row.

CREATE TABLE eval_result (
    trace_id                UUID         NOT NULL REFERENCES trace(trace_id),
    evaluator_id            TEXT         NOT NULL,
    evaluator_version_id    TEXT         NOT NULL REFERENCES evaluator_version(evaluator_version_id),
    dataset_version_id      TEXT         REFERENCES dataset_version(dataset_version_id),   -- NULL for pure online eval
    run_id                  UUID         NOT NULL REFERENCES run(run_id),

    -- Verdict body.
    score                   REAL,                                 -- NULL on error/timeout/over_budget
    label                   TEXT,                                 -- optional categorical outcome
    confidence              REAL,                                 -- [0,1] for probabilistic judges
    reason                  TEXT,                                 -- short free text (judge rationale)
    status                  TEXT         NOT NULL,                -- ok | timeout | error | over_budget | skipped
    latency_ms              INT          NOT NULL,
    cost_usd                NUMERIC(10,6) NOT NULL,

    -- Extension bag. Flat key/value pairs; never used for lineage.
    attributes              JSONB        NOT NULL DEFAULT '{}',

    schema_version          TEXT         NOT NULL,
    first_seen_ns           BIGINT       NOT NULL,
    last_updated_ns         BIGINT       NOT NULL,

    PRIMARY KEY (trace_id, evaluator_id, evaluator_version_id)
);

CREATE INDEX eval_result_by_run         ON eval_result (run_id);
CREATE INDEX eval_result_by_evaluator   ON eval_result (evaluator_id, evaluator_version_id, last_updated_ns);
CREATE INDEX eval_result_by_dataset     ON eval_result (dataset_version_id);
```

### 2.3 `schemas/lineage_dims.sql` — the five dimension tables

```sql
-- app_version: one row per deployable artifact. Content-addressed by the
-- resolved manifest sha (build pin + resolved model ids + feature flags).
CREATE TABLE app_version (
    app_version_id          TEXT         PRIMARY KEY,      -- "sha256:<hex>"
    git_sha                 CHAR(40)     NOT NULL,
    build_tag               TEXT,                          -- human label, optional
    model_pin               JSONB        NOT NULL,         -- {"planner": "gpt-4o-2024-08-06", ...}
    feature_flags           JSONB        NOT NULL,
    created_ns              BIGINT       NOT NULL
);

-- prompt_version: one row per unique prompt template. Content-addressed by
-- the SHA of the normalised template body.
CREATE TABLE prompt_version (
    prompt_version_id       TEXT         PRIMARY KEY,      -- "sha256:<hex>"
    name                    TEXT         NOT NULL,         -- e.g. "support.plan"
    template_sha256         CHAR(64)     NOT NULL UNIQUE,
    template_body           TEXT         NOT NULL,
    variables               JSONB        NOT NULL,         -- declared variable names
    created_ns              BIGINT       NOT NULL
);

-- dataset_version: one row per test-case snapshot. Owned by exercise-02;
-- referenced here to keep the FK story closed.
CREATE TABLE dataset_version (
    dataset_version_id      TEXT         PRIMARY KEY,      -- "sha256:<hex>"
    dataset_name            TEXT         NOT NULL,
    row_count               INT          NOT NULL,
    content_sha256          CHAR(64)     NOT NULL UNIQUE,
    parent_version_id       TEXT         REFERENCES dataset_version(dataset_version_id),
    created_ns              BIGINT       NOT NULL
);

-- evaluator_version: one row per evaluator "release". A judge prompt bump is
-- a new row. A rules change is a new row. Code-only refactors are not.
CREATE TABLE evaluator_version (
    evaluator_version_id    TEXT         PRIMARY KEY,      -- "sha256:<hex>"
    evaluator_id            TEXT         NOT NULL,         -- stable id, e.g. "judge.helpfulness"
    evaluator_type          TEXT         NOT NULL,         -- rule | judge | metric | human
    judge_prompt_sha256     CHAR(64),                      -- NULL for non-judge evaluators
    rubric_sha256           CHAR(64),                      -- for human / judge evaluators
    code_ref                TEXT         NOT NULL,         -- git sha of evaluator code
    created_ns              BIGINT       NOT NULL
);

-- run: one row per batch / stream / CI job. Lets a dashboard group verdicts
-- by "the Thursday regression" or "PR #1247 gate run".
CREATE TABLE run (
    run_id                  UUID         PRIMARY KEY,
    kind                    TEXT         NOT NULL,         -- ci | online | scheduled | ad_hoc | human
    initiator               TEXT         NOT NULL,         -- user or service account
    dataset_version_id      TEXT         REFERENCES dataset_version(dataset_version_id),
    app_version_id          TEXT         NOT NULL REFERENCES app_version(app_version_id),
    started_ns              BIGINT       NOT NULL,
    finished_ns             BIGINT,                        -- NULL while running
    status                  TEXT         NOT NULL,         -- running | ok | failed | aborted
    metadata                JSONB        NOT NULL DEFAULT '{}'
);
```

### 2.4 `schemas/views.sql` — the two read views the UI uses

```sql
-- v_verdict_full: the "one row per verdict, denormalised for humans" view.
-- The UI and the downstream CI gate use this; never the raw table.
CREATE VIEW v_verdict_full AS
SELECT
    er.trace_id,
    t.tenant_id, t.surface, t.tool_shape,
    er.evaluator_id, ev.evaluator_type,
    er.evaluator_version_id, ev.code_ref         AS evaluator_code_ref,
    er.dataset_version_id, d.dataset_name,
    er.run_id, r.kind                            AS run_kind,
    t.app_version_id, a.git_sha                  AS app_git_sha,
    t.prompt_version_id, p.name                  AS prompt_name,
    er.score, er.label, er.confidence, er.reason,
    er.status, er.latency_ms, er.cost_usd,
    er.last_updated_ns
FROM eval_result er
JOIN trace             t ON t.trace_id = er.trace_id
JOIN app_version       a ON a.app_version_id = t.app_version_id
JOIN prompt_version    p ON p.prompt_version_id = t.prompt_version_id
JOIN evaluator_version ev ON ev.evaluator_version_id = er.evaluator_version_id
JOIN run               r ON r.run_id = er.run_id
LEFT JOIN dataset_version d ON d.dataset_version_id = er.dataset_version_id;

-- v_lineage_for_trace: given a trace_id, return every artefact that produced
-- any verdict on it. Used by the "why did this verdict change?" UI.
CREATE VIEW v_lineage_for_trace AS
SELECT
    t.trace_id,
    t.app_version_id, a.git_sha, a.model_pin, a.feature_flags,
    t.prompt_version_id, p.name AS prompt_name, p.template_sha256,
    er.evaluator_id, er.evaluator_version_id, ev.code_ref,
    er.dataset_version_id, d.content_sha256 AS dataset_sha,
    er.run_id, r.kind AS run_kind,
    er.last_updated_ns
FROM trace t
LEFT JOIN eval_result er ON er.trace_id = t.trace_id
JOIN app_version       a ON a.app_version_id = t.app_version_id
JOIN prompt_version    p ON p.prompt_version_id = t.prompt_version_id
LEFT JOIN evaluator_version ev ON ev.evaluator_version_id = er.evaluator_version_id
LEFT JOIN run              r  ON r.run_id = er.run_id
LEFT JOIN dataset_version  d  ON d.dataset_version_id = er.dataset_version_id;
```

### 2.5 `contracts/trace_envelope.json` — the write contract

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "TraceEnvelope",
  "type": "object",
  "required": [
    "trace_id", "tenant_id", "tenant_class", "surface",
    "app_version_id", "prompt_version_id",
    "user_id_hash", "session_id_hash", "redaction_policy_version",
    "started_ns", "duration_ns", "status", "tool_shape",
    "tokens_prompt", "tokens_completion", "cost_usd",
    "payload_uri", "payload_sha256", "schema_version"
  ],
  "properties": {
    "trace_id":                 { "type": "string", "format": "uuid" },
    "tenant_id":                { "type": "string", "minLength": 1 },
    "tenant_class":             { "enum": ["paid", "free", "internal"] },
    "surface":                  { "type": "string", "pattern": "^[a-z0-9_]+$" },
    "app_version_id":           { "type": "string", "pattern": "^sha256:[0-9a-f]{64}$" },
    "prompt_version_id":        { "type": "string", "pattern": "^sha256:[0-9a-f]{64}$" },
    "user_id_hash":             { "type": "string", "pattern": "^[0-9a-f]{16}$" },
    "session_id_hash":          { "type": "string", "pattern": "^[0-9a-f]{16}$" },
    "redaction_policy_version": { "type": "string", "pattern": "^v\\d+\\.\\d+\\.\\d+$" },
    "started_ns":               { "type": "integer", "minimum": 0 },
    "duration_ns":              { "type": "integer", "minimum": 0 },
    "status":                   { "enum": ["ok", "error", "timeout"] },
    "tool_shape":               { "enum": ["chat_only", "tool_call", "rag", "multi_tool"] },
    "tokens_prompt":            { "type": "integer", "minimum": 0 },
    "tokens_completion":        { "type": "integer", "minimum": 0 },
    "cost_usd":                 { "type": "number", "minimum": 0 },
    "payload_uri":              { "type": "string", "pattern": "^s3://|^gs://|^file://" },
    "payload_sha256":           { "type": "string", "pattern": "^[0-9a-f]{64}$" },
    "schema_version":           { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" }
  },
  "additionalProperties": false
}
```

### 2.6 `contracts/eval_result_envelope.json`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "title": "EvalResultEnvelope",
  "type": "object",
  "required": [
    "trace_id", "evaluator_id", "evaluator_version_id", "run_id",
    "status", "latency_ms", "cost_usd", "schema_version"
  ],
  "properties": {
    "trace_id":              { "type": "string", "format": "uuid" },
    "evaluator_id":          { "type": "string", "pattern": "^[a-z0-9_.]+$" },
    "evaluator_version_id":  { "type": "string", "pattern": "^sha256:[0-9a-f]{64}$" },
    "dataset_version_id":    { "type": ["string", "null"], "pattern": "^sha256:[0-9a-f]{64}$" },
    "run_id":                { "type": "string", "format": "uuid" },
    "score":                 { "type": ["number", "null"] },
    "label":                 { "type": ["string", "null"] },
    "confidence":            { "type": ["number", "null"], "minimum": 0, "maximum": 1 },
    "reason":                { "type": ["string", "null"], "maxLength": 2000 },
    "status":                { "enum": ["ok", "timeout", "error", "over_budget", "skipped"] },
    "latency_ms":            { "type": "integer", "minimum": 0 },
    "cost_usd":              { "type": "number", "minimum": 0 },
    "attributes":            { "type": "object" },
    "schema_version":        { "type": "string", "pattern": "^\\d+\\.\\d+\\.\\d+$" }
  },
  "additionalProperties": false,
  "allOf": [
    {
      "if":   { "properties": { "status": { "const": "ok" } } },
      "then": { "required": ["score"] }
    }
  ]
}
```

### 2.7 `docs/schema_versioning.md` — the evolution policy

```markdown
# Schema versioning policy

All trace and eval_result rows carry a `schema_version` column. Readers
MUST switch on the major version; MINOR and PATCH are backwards-compatible.

## Allowed changes by bump

| Bump   | Allowed changes                                                                 | Example                                                           |
|--------|---------------------------------------------------------------------------------|-------------------------------------------------------------------|
| PATCH  | Doc-only, example-only, regex tightening that still accepts prior valid rows.    | Narrow `evaluator_id` regex from `.+` to `^[a-z0-9_.]+$` once nobody writes other shapes. |
| MINOR  | Added optional columns / attributes. New enum values at an existing field.       | Add `confidence` to `eval_result`; add `multi_tool` to `tool_shape`. |
| MAJOR  | Removed columns, renamed columns, type changes, mandatory new columns.           | Replace `user_id` text with `user_id_hash`.                       |

## Deprecation window

A column marked DEPRECATED stays readable for 90 days. Writers are expected
to stop emitting it within 30 days. Readers that still consume it get a
`schema_warning` row in the daily lineage audit (§ exercise-04).

## Rejection at ingest

The ingest workers reject any row where the declared `schema_version` is more
than one MAJOR behind the current contract. This prevents a decommissioned
writer from re-populating a retired column.
```

### 2.8 `pii/redaction_policy_v1.yaml` — the PII boundary

```yaml
# This file is referenced by `redaction_policy_version` on every trace row.
# A new version requires a PR review from security; the `redaction_policy_version`
# column keeps the policy and the row joined for audits.
policy_version: v1.0.0
drop_fields:
  - ssn
  - credit_card
  - passport
  - home_address
hash_fields:                            # keep hash, drop raw
  - user_id
  - session_id
  - email
  - phone
redact_fields:                          # keep shape, replace content with "[REDACTED]"
  - message_text               # user-facing messages
  - tool_arguments.query       # retrieved query strings
retain_for_days: 180
```

### 2.9 `tests/test_envelope_validation.py` — contract tests

```python
import json
import pathlib
import pytest
import jsonschema

ROOT = pathlib.Path(__file__).parent.parent
TRACE_SCHEMA  = json.loads((ROOT / "contracts/trace_envelope.json").read_text())
EVAL_SCHEMA   = json.loads((ROOT / "contracts/eval_result_envelope.json").read_text())


def _valid_trace() -> dict:
    return {
        "trace_id": "01925a5d-2b7e-7f71-9b0c-6b9a3d6a0b23",
        "tenant_id": "acme",
        "tenant_class": "paid",
        "surface": "support_agent",
        "app_version_id": "sha256:" + "a" * 64,
        "prompt_version_id": "sha256:" + "b" * 64,
        "user_id_hash": "0" * 16,
        "session_id_hash": "1" * 16,
        "redaction_policy_version": "v1.0.0",
        "started_ns": 1_700_000_000_000_000_000,
        "duration_ns": 420_000_000,
        "status": "ok",
        "tool_shape": "tool_call",
        "tokens_prompt": 842, "tokens_completion": 117, "cost_usd": 0.00912,
        "payload_uri": "s3://traces/2026/11/15/01925a5d-....parquet",
        "payload_sha256": "c" * 64,
        "schema_version": "1.0.0",
    }


def test_trace_envelope_accepts_canonical_row():
    jsonschema.validate(_valid_trace(), TRACE_SCHEMA)


def test_trace_envelope_rejects_raw_user_id():
    bad = _valid_trace() | {"user_id_hash": "user_12345"}
    with pytest.raises(jsonschema.ValidationError):
        jsonschema.validate(bad, TRACE_SCHEMA)


def test_eval_result_ok_requires_score():
    bad = {
        "trace_id": "01925a5d-2b7e-7f71-9b0c-6b9a3d6a0b23",
        "evaluator_id": "judge.helpfulness",
        "evaluator_version_id": "sha256:" + "d" * 64,
        "run_id": "01925a5d-aaaa-7777-bbbb-cccccccccccc",
        "status": "ok",
        "latency_ms": 420, "cost_usd": 0.0021,
        "schema_version": "1.0.0",
    }
    with pytest.raises(jsonschema.ValidationError):
        jsonschema.validate(bad, EVAL_SCHEMA)


def test_eval_result_over_budget_may_have_null_score():
    row = {
        "trace_id": "01925a5d-2b7e-7f71-9b0c-6b9a3d6a0b23",
        "evaluator_id": "judge.helpfulness",
        "evaluator_version_id": "sha256:" + "d" * 64,
        "run_id": "01925a5d-aaaa-7777-bbbb-cccccccccccc",
        "score": None, "status": "over_budget",
        "latency_ms": 0, "cost_usd": 0.0,
        "schema_version": "1.0.0",
    }
    jsonschema.validate(row, EVAL_SCHEMA)
```

### 2.10 Worked lineage query — "why did this verdict change?"

```sql
-- Given a trace_id, surface everything that could have moved the verdict:
-- app version, prompt version, evaluator version, dataset version, run kind.
-- Join once; the UI renders the diff client-side.
SELECT
    trace_id,
    app_version_id,     prompt_version_id,
    evaluator_id,       evaluator_version_id,
    dataset_version_id,
    run_id,             run_kind,
    last_updated_ns
FROM v_lineage_for_trace
WHERE trace_id = $1
ORDER BY last_updated_ns;
```

The read is cheap: each dimension is a content-addressed FK, and a diff
between two runs for the same `trace_id` boils down to which hash changed.

## 3. Validation steps

1. **Load the DDL into a local Postgres**: `psql -f schemas/trace.sql -f schemas/eval_result.sql -f schemas/lineage_dims.sql -f schemas/views.sql`. No errors.
2. **Insert a canonical fixture**: a seed script writes two `app_version`, two `prompt_version`, one `dataset_version`, two `evaluator_version`, one `run`, 100 `trace`, and 300 `eval_result` rows. All FKs resolve.
3. **Run the envelope contract tests**: `pytest -q tests/test_envelope_validation.py`. All assertions pass; a `user_id_hash` of `"user_12345"` is rejected; an `ok` status without a score is rejected; an `over_budget` row with null score is accepted.
4. **Query the lineage view**: `SELECT * FROM v_lineage_for_trace WHERE trace_id = ...` returns one row per `(evaluator, evaluator_version)`; every column is populated except `dataset_version_id` for pure online rows.
5. **Attempt a bad write**: an `eval_result` row missing `evaluator_version_id` or `run_id` fails with a NOT NULL violation. An `app_version_id` that does not exist fails with an FK violation.
6. **Replay a verdict**: write the same `(trace_id, evaluator_id, evaluator_version_id)` twice. The PK conflict either upserts (if the writer uses `ON CONFLICT`) or rejects. No duplicate rows.
7. **Bump an evaluator**: write a new `evaluator_version` row for the same `evaluator_id` and replay 100 traces; both versions now coexist in `eval_result`. `SELECT COUNT(*), evaluator_version_id FROM eval_result GROUP BY 2` shows two populated versions.
8. **PII audit**: `SELECT COUNT(*) FROM trace WHERE user_id_hash ~ '^u_' OR length(user_id_hash) != 16;` returns zero.
9. **Schema version check**: inserting a row with `schema_version='0.9.0'` is accepted (minor behind); inserting `schema_version='0.0.1'` is rejected per §2.7.

## 4. Rubric / review checklist

| #   | Criterion                                                                                                    | Pass                                                                                     | Fail                                                            |
| --- | ------------------------------------------------------------------------------------------------------------ | ---------------------------------------------------------------------------------------- | --------------------------------------------------------------- |
| R1  | `trace` carries foreign keys to `app_version` and `prompt_version`; neither is nullable.                      | DDL marks both `NOT NULL REFERENCES`.                                                    | Lineage lives in a JSON blob or is nullable.                   |
| R2  | `eval_result` primary key includes `evaluator_version_id`; bumping an evaluator does not overwrite history.    | PK = `(trace_id, evaluator_id, evaluator_version_id)`; test proves two versions coexist. | PK omits version; new version silently overwrites.              |
| R3  | Every content-addressed id is a SHA (`sha256:<hex>`), not a human label.                                      | `app_version_id`, `prompt_version_id`, etc., validated by regex in envelope schema.      | Free-text ids like `"v2"` accepted.                              |
| R4  | PII never lands raw. `trace` stores hashes with a `redaction_policy_version`.                                  | DDL has `user_id_hash` / `session_id_hash` / `redaction_policy_version` columns.          | Raw `user_id` present; no policy linkage.                       |
| R5  | `status` is a first-class enum including `timeout`, `error`, `over_budget`; evaluator rows are written in every case. | Status enum defined; envelope allows null score when status ≠ ok.                         | Timeouts / budget-exhausted rows are dropped.                   |
| R6  | `schema_version` is a column on both fact tables with a documented evolution policy (§2.7).                    | Column present; `docs/schema_versioning.md` defines minor/major rules and deprecation window. | Version only in docs; no column.                                |
| R7  | `run` ties every verdict to the batch / stream / CI job that produced it.                                     | `eval_result.run_id` NOT NULL REFERENCES run.                                             | Run id only in attributes; no FK.                               |
| R8  | A read view denormalises lineage for the UI (`v_lineage_for_trace`) so the "why did it change?" query is one join. | View defined and tested.                                                                 | UI expected to join seven tables ad hoc.                        |
| R9  | Envelope JSON Schemas reject malformed writes at the API boundary.                                             | Contract tests prove rejection paths.                                                    | Schemas missing or only documentary.                            |
| R10 | Content hashes collapse duplicates. Two prompts with the same body get one `prompt_version_id`.                | `template_sha256` has UNIQUE index; test inserts twice and sees one row.                  | Each write produces a new version id regardless of content.      |

A passing submission is 8+ of 10 with R1, R2, R4, R5, R6 all hard-required.

## 5. Common mistakes

- **Lineage in a JSON blob.** Storing `app_version` as a free-text key inside `attributes` means the ingest can't reject missing lineage and the dashboard can't JOIN. Promote every lineage field to its own foreign-keyed column.
- **Evaluator version as a column, not a key.** When a judge prompt changes, you want the old verdicts to stay queryable so regression analysis can compare. Putting the version in the primary key enforces coexistence; putting it in a column silently overwrites.
- **Free-text version labels.** Humans call two different templates "v2". Content-address every artefact with a SHA so equality is decidable and the PK does not lie.
- **Raw PII in the eval plane.** Even hashed, the hashes should be produced upstream, and the policy that produced them should be named on the row. This is the NIST AI RMF `MAP 1.1` boundary and OWASP ML Top 10 `ML02` / MITRE ATLAS `AML.T0024` control point.
- **At-most-once writes.** Dropping a verdict on timeout corrupts downstream coverage math (exercise-04 SLOs, mod-107 confidence sequences). Always write a row with the right `status`.
- **Missing schema version.** Skipping `schema_version` makes the first migration a database-wide outage. Pay the one column of cost upfront.
- **One giant "metadata" bag.** Pushing feature flags, model pins, and dataset ids into one JSON column loses type safety and makes indexing impossible. The five dimension tables cost a day of work and buy years of clean lineage queries.
- **Overloading the trace table.** Putting judge scores and span bodies on the trace row breaks the append-mostly invariant. Verdicts belong on `eval_result`; span bodies belong in object storage.
- **No `run_id`.** Without `run_id`, you can't answer "show me the regression suite verdicts for last Thursday" without reconstructing time ranges by hand.
- **Ignoring tool_shape in the trace row.** The tool-shape dimension is the single most useful axis for slicing drift and coverage; promoting it to a column pays for itself the first week.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `GOVERN 1.6` (traceability), `MAP 1.1` (data lineage), `MEASURE 2.3` (metric coverage).
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning) motivates the PII boundary at the ingest layer, `ML07` (transfer learning attack) motivates artefact hashing.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0024` (exfiltration via LLM) and `AML.M0015` (prompt hardening) frame why PII redaction and policy versioning are first-class.
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/>, for the span attributes mirrored on the trace row.
- JSON Schema draft 2020-12, <https://json-schema.org/draft/2020-12/schema>, for the envelope contracts.
- Local exercise reader: `lessons/mod-110-eval-data-platform-slice/exercises/exercise-01-trace-and-eval-result-schema-with-lineage.md`.
- Related solutions in this module: exercise-02 owns `dataset_version`; exercise-03 owns `run`; exercise-04 reads `v_verdict_full` for SLOs; exercise-05 writes `run.kind='ci'` rows.
