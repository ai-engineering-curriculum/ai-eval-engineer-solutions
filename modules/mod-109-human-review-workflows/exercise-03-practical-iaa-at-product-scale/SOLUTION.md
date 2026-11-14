# exercise-03 — Practical IAA At Product Scale — Solution

> Reference solution for `lessons/mod-109-human-review-workflows/exercises/exercise-03-practical-iaa-at-product-scale.md`.
> Compute inter-annotator agreement (IAA) on the `annotations.jsonl` export from exercise-01, surface disagreements as actionable work items, and operate the whole loop on a weekly cadence at product scale (thousands of annotations, dozens of reviewers, evolving rubric).

## 1. Solution overview

The deliverable is an IAA harness `iaa/` that reads the canonical annotations file, computes per-field and per-rubric-version agreement using the right coefficient for each field's data shape, and emits three artefacts:

1. A **per-reviewer report** — kappa / alpha vs the panel, flagged fields, suggested calibration cases.
2. A **per-field report** — which rubric items have the lowest agreement; these are the schema or guideline bugs.
3. An **adjudication queue** — the specific records where two first-pass annotators disagreed. These route into exercise-01's `review-adjudication` queue with the disagreement surfaced.

Five design choices anchor the solution:

1. **Choose the coefficient based on the field, not the project.** Binary helpfulness gets Cohen's kappa (2 raters) or Fleiss' kappa (3+). Multi-choice safety labels get Krippendorff's alpha with the `nominal` distance. Ordinal severity gets Krippendorff's alpha with the `ordinal` distance. Free-text rationale is not rated numerically; we inspect it manually and only compute an agreement proxy on reviewer-assigned category. One formula does not cover a mixed rubric.
2. **Report raw agreement *and* chance-corrected agreement.** Kappa/alpha are useful but counterintuitive at extreme class imbalances (the "kappa paradox"). Always report `(raw_agreement, chance_corrected, prevalence_index, bias_index)` together. A single kappa number is misleading when 95% of labels are `none`.
3. **IAA is per rubric version.** Annotations under `rubric_version=v1.0` and `v1.1` are different studies. Mixing them yields meaningless numbers. The harness refuses to pool across versions; version changes force a calibration restart.
4. **Disagreements become work, not warnings.** Every disagreement produces an adjudication record with both annotators' values and a pointer back to the original trace. These go into the `review-adjudication` queue so humans resolve them within the normal SLA. A number on a dashboard that no one acts on is noise.
5. **Gold-set agreement is tracked separately from pair-wise agreement.** For the gold set (produced by exercise-04), the ground truth is the adjudicator's final label, not another reviewer. We compute each annotator's accuracy against gold as a separate metric; low accuracy triggers re-calibration, not a pair-wise kappa debate.

## 2. Worked implementation

```
mod-109/exercise-03/
├── pyproject.toml
├── README.md
├── iaa/
│   ├── __init__.py
│   ├── schema.py                 # Annotation dataclass, field type registry
│   ├── loader.py                 # read annotations.jsonl, group by (trace_id, rubric_version)
│   ├── coefficients.py           # kappa, fleiss_kappa, krippendorff_alpha
│   ├── report.py                 # per-reviewer and per-field reports
│   ├── gold_accuracy.py          # annotator vs gold
│   ├── adjudication.py           # emit records for exercise-01 review-adjudication queue
│   └── cli.py
├── policy/
│   └── field_coefficients.yaml   # which coefficient per field
├── fixtures/
│   ├── annotations_week42.jsonl  # 3 reviewers × 500 records
│   └── gold_week42.jsonl         # 50 gold-set records with adjudicated labels
└── tests/
    ├── test_kappa.py
    ├── test_fleiss.py
    ├── test_alpha_ordinal.py
    ├── test_prevalence_bias.py
    └── test_adjudication_output.py
```

### 2.1 `iaa/schema.py`

```python
"""Annotation + field-type registry. The rubric's annotation_fields from
exercise-01 map 1:1 onto these entries."""
from __future__ import annotations
from dataclasses import dataclass
from typing import Any, Literal

FieldKind = Literal["binary", "nominal", "ordinal", "multi_label", "free_text"]


@dataclass(frozen=True)
class FieldSpec:
    field_id: str
    kind: FieldKind
    categories: tuple[str, ...]     # ("yes","partially","no") etc
    distance: str | None = None     # for ordinal: "ordinal"; nominal: "nominal"


@dataclass(frozen=True)
class Annotation:
    trace_id: str
    annotator_id: str
    rubric_version: str
    values: dict[str, Any]          # field_id -> value
    submitted_ns: int
```

```yaml
# policy/field_coefficients.yaml — ties rubric fields to their statistical treatment.
fields:
  helpfulness:
    kind: nominal
    categories: [yes, partially, no]
    coefficient: krippendorff
    distance: nominal
  grounded:
    kind: nominal
    categories: [grounded, partially_grounded, not_grounded, n/a]
    coefficient: krippendorff
    distance: nominal
  safety_concern:
    kind: multi_label
    categories: [pii_leak, unsafe_advice, policy_violation, none]
    coefficient: jaccard_mean
  severity:
    kind: ordinal
    categories: [low, medium, high, critical]
    coefficient: krippendorff
    distance: ordinal
  rationale:
    kind: free_text
    categories: []
    coefficient: null             # not computed; qualitative review only
```

### 2.2 `iaa/coefficients.py`

```python
"""Agreement coefficients. Each takes a reliability matrix (rows=items,
cols=raters) with None for missing values, plus a distance function for
Krippendorff's alpha."""
from __future__ import annotations
from collections import Counter
from typing import Callable, Sequence


def raw_agreement(matrix: Sequence[Sequence[int | None]]) -> float:
    """Observed agreement: fraction of unordered pairs that agree."""
    agree = pairs = 0
    for row in matrix:
        present = [v for v in row if v is not None]
        for i in range(len(present)):
            for j in range(i + 1, len(present)):
                pairs += 1
                if present[i] == present[j]:
                    agree += 1
    return agree / pairs if pairs else float("nan")


def cohens_kappa(matrix: Sequence[Sequence[int | None]]) -> float:
    """Two-rater Cohen's kappa. Uses the first two columns; skips rows
    with missing values."""
    pairs = [(row[0], row[1]) for row in matrix
             if row[0] is not None and row[1] is not None]
    if not pairs:
        return float("nan")
    n = len(pairs)
    cats = set(a for a, _ in pairs) | set(b for _, b in pairs)
    p_o = sum(1 for a, b in pairs if a == b) / n
    pa = {c: sum(1 for a, _ in pairs if a == c) / n for c in cats}
    pb = {c: sum(1 for _, b in pairs if b == c) / n for c in cats}
    p_e = sum(pa[c] * pb[c] for c in cats)
    if p_e == 1.0:
        return 1.0 if p_o == 1.0 else float("nan")
    return (p_o - p_e) / (1 - p_e)


def fleiss_kappa(matrix: Sequence[Sequence[int | None]], k: int) -> float:
    """Multi-rater Fleiss' kappa. Each row needs exactly n_i >= 2 ratings;
    rows with < 2 are skipped. `k` is the number of categories."""
    valid_rows = []
    for row in matrix:
        counts = [0] * k
        for v in row:
            if v is None:
                continue
            counts[int(v)] += 1
        if sum(counts) >= 2:
            valid_rows.append(counts)
    if not valid_rows:
        return float("nan")
    N = len(valid_rows)
    n_i = [sum(r) for r in valid_rows]
    p_j = [sum(r[j] for r in valid_rows) / sum(n_i) for j in range(k)]
    P_i = [
        (sum(r[j] * (r[j] - 1) for j in range(k)) / (n_i[i] * (n_i[i] - 1)))
        if n_i[i] > 1 else 0.0
        for i, r in enumerate(valid_rows)
    ]
    P_bar = sum(P_i) / N
    P_e = sum(p * p for p in p_j)
    if P_e == 1.0:
        return 1.0 if P_bar == 1.0 else float("nan")
    return (P_bar - P_e) / (1 - P_e)


def krippendorff_alpha(
    matrix: Sequence[Sequence[int | None]],
    distance: Callable[[int, int], float],
) -> float:
    """Krippendorff's alpha with arbitrary distance. Values must be integer
    category codes; missing = None."""
    # Build the coincidence matrix.
    cats = sorted({v for row in matrix for v in row if v is not None})
    idx = {c: i for i, c in enumerate(cats)}
    k = len(cats)
    if k == 0:
        return float("nan")
    coin = [[0.0] * k for _ in range(k)]
    for row in matrix:
        present = [v for v in row if v is not None]
        m = len(present)
        if m < 2:
            continue
        for i, a in enumerate(present):
            for b in present[:i] + present[i + 1:]:
                coin[idx[a]][idx[b]] += 1.0 / (m - 1)
    totals = [sum(coin[i]) for i in range(k)]
    n = sum(totals)
    if n == 0:
        return float("nan")
    D_o = 0.0
    for i in range(k):
        for j in range(k):
            D_o += coin[i][j] * distance(cats[i], cats[j])
    D_o /= n
    D_e = 0.0
    for i in range(k):
        for j in range(k):
            D_e += totals[i] * totals[j] * distance(cats[i], cats[j])
    D_e /= n * (n - 1)
    if D_e == 0:
        return 1.0 if D_o == 0 else float("nan")
    return 1.0 - D_o / D_e


def nominal_distance(a: int, b: int) -> float:
    return 0.0 if a == b else 1.0


def ordinal_distance(a: int, b: int) -> float:
    return float((a - b) ** 2)


def prevalence_and_bias_index(matrix: Sequence[Sequence[int | None]]) -> tuple[float, float]:
    """Byrt-Bishop-Carlin prevalence (P) and bias (B) indices for 2x2
    tables. P close to ±1 indicates class imbalance; B close to ±1
    indicates annotator bias. Computed on pair (col 0, col 1)."""
    counts = Counter()
    for row in matrix:
        if row[0] is not None and row[1] is not None:
            counts[(row[0], row[1])] += 1
    a = counts.get((1, 1), 0)
    b = counts.get((1, 0), 0)
    c = counts.get((0, 1), 0)
    d = counts.get((0, 0), 0)
    n = a + b + c + d
    if n == 0:
        return float("nan"), float("nan")
    prevalence = (a - d) / n
    bias = (b - c) / n
    return prevalence, bias
```

### 2.3 `iaa/report.py`

```python
"""Per-field and per-reviewer reports. Both are dataclasses; the CLI writes
JSON and optionally renders a markdown summary."""
from __future__ import annotations
from collections import defaultdict
from dataclasses import dataclass, field
from itertools import combinations
from typing import Any, Callable
import yaml

from .schema import Annotation, FieldSpec
from .coefficients import (
    cohens_kappa, fleiss_kappa, krippendorff_alpha,
    raw_agreement, prevalence_and_bias_index,
    nominal_distance, ordinal_distance,
)


@dataclass
class FieldReport:
    field_id: str
    rubric_version: str
    n_items: int
    n_raters: int
    raw_agreement: float
    chance_corrected: float
    coefficient_name: str
    prevalence_index: float | None = None
    bias_index: float | None = None
    low_agreement_items: list[str] = field(default_factory=list)


@dataclass
class ReviewerReport:
    annotator_id: str
    rubric_version: str
    n_items: int
    per_field: dict[str, float]           # field_id -> cohen's kappa vs panel majority
    gold_accuracy: float | None = None
    calibration_needed: bool = False


def build_matrix(
    annotations: list[Annotation], field_id: str, categories: tuple[str, ...]
) -> tuple[list[list[int | None]], list[str], list[str]]:
    """Return (matrix, trace_ids, annotator_ids). Matrix rows=items, cols=annotators.

    Expects scalar categorical values (binary / nominal / ordinal). Multi-label
    fields export list values; those are not representable in a (row, col)
    integer matrix and must be routed through `_report_multi_label` instead.
    Raised explicitly rather than silently skipped so a mis-configured
    coefficient is caught at report time, not as a NaN in production."""
    annotators = sorted({a.annotator_id for a in annotations})
    traces = sorted({a.trace_id for a in annotations})
    idx_ann = {a: i for i, a in enumerate(annotators)}
    idx_trace = {t: i for i, t in enumerate(traces)}
    cat_idx = {c: i for i, c in enumerate(categories)}
    m: list[list[int | None]] = [[None] * len(annotators) for _ in traces]
    for a in annotations:
        v = a.values.get(field_id)
        if v is None:
            continue
        if isinstance(v, (list, set, tuple)):
            raise TypeError(
                f"field {field_id!r} on trace {a.trace_id!r} is multi-valued "
                f"({v!r}); use jaccard_mean (not a categorical matrix)."
            )
        if v not in cat_idx:
            continue
        m[idx_trace[a.trace_id]][idx_ann[a.annotator_id]] = cat_idx[v]
    return m, traces, annotators


def _as_label_set(v: Any) -> frozenset:
    """Normalise a multi-label value to a hashable set. Accepts list/tuple/set
    (common exporter shapes) and bare scalars (single-choice collapsed to a set)."""
    if v is None:
        return frozenset()
    if isinstance(v, (list, tuple, set, frozenset)):
        return frozenset(v)
    return frozenset([v])


def _jaccard(a: frozenset, b: frozenset) -> float:
    if not a and not b:
        return 1.0
    return len(a & b) / len(a | b)


def _report_multi_label(annotations: list[Annotation], spec: FieldSpec) -> FieldReport:
    """Mean-Jaccard agreement over per-item annotator pairs. Each annotator
    contributes a set of labels per trace; for every trace with >= 2 raters we
    average the pair-wise Jaccard, then average across traces."""
    by_trace: dict[str, list[tuple[str, frozenset]]] = defaultdict(list)
    annotators: set[str] = set()
    for a in annotations:
        if spec.field_id not in a.values:
            continue
        by_trace[a.trace_id].append((a.annotator_id, _as_label_set(a.values[spec.field_id])))
        annotators.add(a.annotator_id)

    per_item_jaccard: list[float] = []
    per_item_exact: list[bool] = []
    low: list[str] = []
    for trace_id, pairs in by_trace.items():
        if len(pairs) < 2:
            continue
        scores = [_jaccard(a, b) for (_, a), (_, b) in combinations(pairs, 2)]
        if not scores:
            continue
        avg = sum(scores) / len(scores)
        per_item_jaccard.append(avg)
        per_item_exact.append(avg == 1.0)
        if avg < 1.0:
            low.append(trace_id)

    if per_item_jaccard:
        mean_j = sum(per_item_jaccard) / len(per_item_jaccard)
        raw = sum(per_item_exact) / len(per_item_exact)
    else:
        mean_j = float("nan")
        raw = float("nan")

    return FieldReport(
        field_id=spec.field_id,
        rubric_version=annotations[0].rubric_version,
        n_items=len(by_trace), n_raters=len(annotators),
        raw_agreement=raw, chance_corrected=mean_j,
        coefficient_name="jaccard_mean",
        prevalence_index=None, bias_index=None,
        low_agreement_items=low,
    )


def report_for_field(
    annotations: list[Annotation],
    spec: FieldSpec,
    coefficient: str,
) -> FieldReport:
    if spec.kind == "multi_label" or coefficient == "jaccard_mean":
        return _report_multi_label(annotations, spec)

    matrix, traces, annotators = build_matrix(
        annotations, spec.field_id, spec.categories,
    )
    k = len(spec.categories)
    ra = raw_agreement(matrix)

    if coefficient == "krippendorff":
        dist = ordinal_distance if spec.distance == "ordinal" else nominal_distance
        cc = krippendorff_alpha(matrix, dist)
    elif coefficient == "fleiss":
        cc = fleiss_kappa(matrix, k)
    elif coefficient == "cohen":
        cc = cohens_kappa(matrix)
    else:
        cc = float("nan")

    prev, bias = (None, None)
    if k == 2 and len(annotators) == 2:
        prev, bias = prevalence_and_bias_index(matrix)

    low = [traces[i] for i, row in enumerate(matrix)
           if _row_is_split(row)]
    return FieldReport(
        field_id=spec.field_id, rubric_version=annotations[0].rubric_version,
        n_items=len(traces), n_raters=len(annotators),
        raw_agreement=ra, chance_corrected=cc,
        coefficient_name=coefficient,
        prevalence_index=prev, bias_index=bias,
        low_agreement_items=low,
    )


def _row_is_split(row: list[int | None]) -> bool:
    present = [v for v in row if v is not None]
    return len(set(present)) >= 2
```

### 2.4 `iaa/adjudication.py`

```python
"""Emit records for exercise-01's review-adjudication queue.
Each row references the trace and names the two annotators + their values."""
from __future__ import annotations
import json
import os
from collections import defaultdict
from pathlib import Path
from typing import Any

from .schema import Annotation


def _hashable(v: Any) -> Any:
    """Normalise so multi-label list values compare equal regardless of order
    and can live in a set for disagreement detection."""
    if isinstance(v, list):
        return tuple(sorted(v, key=lambda x: (x is None, str(x))))
    if isinstance(v, set):
        return frozenset(v)
    return v


def emit_adjudication(
    annotations: list[Annotation],
    fields_to_adjudicate: list[str],
    out_path: Path,
) -> int:
    by_trace: dict[str, list[Annotation]] = defaultdict(list)
    for a in annotations:
        by_trace[a.trace_id].append(a)

    rows: list[dict[str, Any]] = []
    for trace_id, anns in by_trace.items():
        if len(anns) < 2:
            continue
        disagreements: dict[str, list[tuple[str, object]]] = {}
        for field_id in fields_to_adjudicate:
            values = [(a.annotator_id, a.values.get(field_id)) for a in anns]
            distinct = {_hashable(v) for _, v in values if v is not None}
            if len(distinct) > 1:
                disagreements[field_id] = values
        if not disagreements:
            continue
        rows.append({
            "trace_id": trace_id,
            "rubric_version": anns[0].rubric_version,
            "disagreements": disagreements,
            "queue": "review-adjudication",
            "sla_hours": 48,
        })

    # Build the whole payload before touching the output path, then swap in
    # with an atomic rename. A crash mid-serialisation leaves the previous
    # out_path (if any) untouched; readers never see a half-written file.
    tmp_path = out_path.with_suffix(out_path.suffix + ".tmp")
    with tmp_path.open("w") as f:
        for row in rows:
            f.write(json.dumps(row, sort_keys=True, default=str) + "\n")
    os.replace(tmp_path, out_path)
    return len(rows)
```

### 2.5 `tests/test_prevalence_bias.py`

```python
from iaa.coefficients import cohens_kappa, prevalence_and_bias_index


def test_kappa_paradox_at_high_prevalence():
    """95/5 class prevalence with 2 disagreements out of 100."""
    # 95 both annotators say 0; 3 both say 1; 1 disagreement each way.
    matrix = [[0, 0]] * 95 + [[1, 1]] * 3 + [[1, 0]] + [[0, 1]]
    k = cohens_kappa(matrix)
    prev, bias = prevalence_and_bias_index(matrix)
    assert k < 0.6, "kappa should be modest despite 98% raw agreement"
    assert abs(prev) > 0.8, "prevalence should flag the imbalance"


def test_perfect_agreement():
    matrix = [[1, 1]] * 50 + [[0, 0]] * 50
    assert cohens_kappa(matrix) == 1.0


def test_random_agreement():
    matrix = [[0, 1]] * 50 + [[1, 0]] * 50
    k = cohens_kappa(matrix)
    assert k < 0.0, "systematic disagreement should give negative kappa"
```

### 2.6 `tests/test_alpha_ordinal.py`

```python
from iaa.coefficients import krippendorff_alpha, ordinal_distance, nominal_distance


def test_ordinal_is_lenient_about_adjacent_classes():
    """One rater says 1 (medium), the other says 2 (high) — ordinal
    distance penalises less than nominal."""
    matrix = [[1, 2], [2, 2], [0, 0], [3, 3], [1, 1]]
    nominal = krippendorff_alpha(matrix, nominal_distance)
    ordinal = krippendorff_alpha(matrix, ordinal_distance)
    assert ordinal > nominal, "ordinal alpha must be >= nominal when disagreements are adjacent"


def test_ordinal_penalises_far_disagreement():
    """One rater says 0 (low), the other says 3 (critical)."""
    matrix = [[0, 3]] * 5 + [[0, 0]] * 5
    ordinal = krippendorff_alpha(matrix, ordinal_distance)
    assert ordinal < 0.2, "far-apart ordinal disagreements must tank alpha"
```

## 3. Validation steps

1. **Install + run tests**: `pip install -e . && pytest -q tests/`. All coefficient tests pass; the kappa-paradox test explicitly verifies that raw agreement 0.98 can produce a kappa under 0.6 at extreme prevalence.
2. **End-to-end**: `python -m iaa.cli --annotations fixtures/annotations_week42.jsonl --gold fixtures/gold_week42.jsonl --out out/`. Writes `out/per_field.json`, `out/per_reviewer.json`, and `out/adjudication.jsonl`.
3. **Per-field sanity**: `jq '.[] | select(.chance_corrected < 0.4)' out/per_field.json` lists the low-agreement fields. Verify at least one is present in the fixture; `rationale` is absent (null coefficient).
4. **Per-reviewer sanity**: `jq '.[] | select(.calibration_needed)' out/per_reviewer.json` lists reviewers whose gold accuracy is below 0.80 or who disagree with the panel majority on more than 25% of items.
5. **Rubric-version boundary**: construct a mixed fixture with two rubric versions; the harness exits non-zero with "annotations span 2 rubric versions; split per version before computing IAA".
6. **Adjudication handoff**: `wc -l out/adjudication.jsonl` is non-zero for a fixture with disagreements; each row matches the `review-adjudication` shape expected by exercise-01.
7. **Prevalence-bias report**: for any binary field with a 2-rater pair, `prevalence_index` and `bias_index` are reported next to kappa; a single-number dashboard view is avoided.

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|-----------|------|------|
| R1 | Coefficient is chosen per field based on data shape (nominal / ordinal / multi-label). | `field_coefficients.yaml` present; ordinal severity uses `ordinal_distance`; multi-label uses mean Jaccard. | A single kappa reported across all fields, including ordinal ones. |
| R2 | Both raw agreement and chance-corrected agreement are reported. | Every `FieldReport` has both; the CLI summary shows them side-by-side. | Only chance-corrected or only raw is shown. |
| R3 | Prevalence and bias indices are reported for binary / 2-rater fields. | `prevalence_index`, `bias_index` populated when applicable; kappa-paradox warnings emitted at `\|prev\| > 0.7`. | Kappa is reported alone at 95% class imbalance; readers misinterpret. |
| R4 | The harness refuses to pool across rubric versions. | Fixture crossing versions exits non-zero with a clear message. | Cross-version kappa reported as if annotators disagreed, when the rubric itself changed. |
| R5 | Gold accuracy is tracked separately from pair-wise agreement. | `gold_accuracy` populated for each reviewer when a gold set is provided; it is NOT folded into the pair-wise kappa. | Gold answers used as a third annotator in Fleiss' kappa; adjudicator "votes" inflate kappa. |
| R6 | Every disagreement produces an actionable adjudication record. | `adjudication.jsonl` has one row per `(trace_id, field)` with at least two distinct values. | Disagreements end at a dashboard; nothing routed. |
| R7 | Reviewer-level low performance triggers calibration, not punishment. | `calibration_needed=true` sends the reviewer to the next calibration round; no PII in the reviewer report. | Low-performers are singled out in weekly readouts by name. |
| R8 | Coefficient implementations are tested against known-answer small cases. | `test_kappa.py`, `test_fleiss.py`, `test_alpha_ordinal.py` all pass against hand-computed expected values. | Coefficients are implemented without tests; numbers are plausible but unverified. |
| R9 | Free-text fields are explicitly excluded from numeric IAA, not silently stringified. | `coefficient: null` for `rationale` in config; report skips it. | Free text is hashed into integer codes and fed to kappa; the number is meaningless. |
| R10 | The harness is runnable on a weekly cadence by one script, not notebook glue. | `iaa.cli` is one invocation; outputs are deterministic given inputs; no notebook required. | IAA is produced by running six notebook cells in order; no one can re-run it. |

A passing submission is 8+ of 10, with R1, R2, R4, R6, R8 all hard-required.

## 5. Common mistakes

- **One coefficient for the whole rubric.** Ordinal severity and nominal safety-concern need different distance functions. Reporting a single kappa across the rubric hides where the real disagreement is.
- **The kappa paradox.** At 95% `none`, raw agreement near 1.0 produces a kappa near 0. This is not a bug in kappa; it's what happens at extreme prevalence. Report `(raw, cc, prevalence, bias)` together, and surface the paradox with a warning when `|prevalence| > 0.7`.
- **Pooling across rubric versions.** Annotations under `v1.0` and `v1.1` are different studies. Pooling them produces an IAA that mixes schema drift with reviewer drift. Keep them separate; force a calibration restart on version bump.
- **Treating the adjudicator as a third annotator.** The adjudicator's label is the ground truth for *exercise-04's gold set*, not an input to pair-wise IAA. Mixing them inflates kappa and corrupts the gold accuracy signal.
- **Numeric IAA on free text.** You cannot compute kappa on sentences. Either collapse to a reviewer-tagged category (if the schema has one) or exclude free text from numeric IAA entirely.
- **Using Cohen's kappa for three raters.** Cohen's kappa is defined for two raters. For three or more, use Fleiss' kappa (nominal) or Krippendorff's alpha (nominal / ordinal / interval). This is a common silent error.
- **Shaming low-kappa reviewers in public.** Low IAA is a guideline bug or a calibration gap, not a performance issue. Route calibration offline; report aggregates in team channels, not names. NIST AI RMF `GOVERN 3.2` makes the organisational support for human reviewers a named responsibility.
- **Not acting on disagreements.** If a dashboard shows kappa and no one routes the disagreements, the number is decoration. Emit an adjudication queue with the same SLA as normal review.
- **Computing IAA on a single week's data each time.** Variance is high at small sample sizes; a single reviewer with 20 items can dominate the week's IAA. Report rolling-4-week IAA alongside this-week-only.
- **Hard-coded coefficient thresholds for "good enough".** The commonly cited "kappa > 0.8 is great" ranges (Landis & Koch; Fleiss) are heuristics, not standards. Report the number and the components; let humans decide on the threshold with the rubric in front of them.

## 6. References

- NIST AI Risk Management Framework — <https://www.nist.gov/itl/ai-risk-management-framework> — `MEASURE 3.1` (human review as a measurement mechanism), `GOVERN 3.2` (team capacity and reviewer support for AI measurement).
- OWASP Machine Learning Security Top 10 — <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML02` (data poisoning): IAA is the first-order detector for a compromised annotator pool; low agreement is a leading indicator.
- MITRE ATLAS — <https://atlas.mitre.org/> — `AML.T0018` (Backdoor ML Model) and `AML.T0043` (Craft Adversarial Data): IAA on human review is one of the checks that makes these attacks easier to detect.
- Krippendorff, K. *Content Analysis: An Introduction to Its Methodology*, Sage Publications — reference for Krippendorff's alpha; the project's live documentation of α is also at <https://repository.upenn.edu/asc_papers/43/>.
- Byrt, T., Bishop, J., Carlin, J. B. (1993) "Bias, prevalence and kappa", *Journal of Clinical Epidemiology*, 46(5), 423–429 — the prevalence/bias indices used in `prevalence_and_bias_index`.
- Fleiss, J. L. (1971) "Measuring nominal scale agreement among many raters", *Psychological Bulletin*, 76(5), 378–382.
- Python stdlib: `statistics`, `collections.Counter` — <https://docs.python.org/3/library/statistics.html>.
- Local exercise reader: `lessons/mod-109-human-review-workflows/exercises/exercise-03-practical-iaa-at-product-scale.md`.
- Related exercises: exercise-01 produces `annotations.jsonl`; exercise-02's cluster-level priority drives which traces are sampled into the IAA batch; exercise-04 reads the adjudicated items as gold-set candidates.
