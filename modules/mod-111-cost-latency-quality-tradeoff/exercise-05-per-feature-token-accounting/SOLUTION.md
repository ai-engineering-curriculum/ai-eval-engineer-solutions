# exercise-05 — Per-Feature Token Accounting — Solution

> Reference solution for `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-05-per-feature-token-accounting.md`.
> This is an **implementation** exercise. The deliverable is a per-feature token accounting system: given a stream of traces, attribute prompt/completion/cached tokens — and the resulting USD cost — to the feature (surface + tool_shape), the tenant, and the model that produced them. The output drives finance chargebacks, "most expensive feature" reports, and the cost half of the multi-objective report (exercise-01). Correctness requires caring about cached tokens, batch vs streaming, and the split between first-call tokens and tool-call tokens.

## 1. Solution overview

Token accounting that is only "sum `gen_ai.usage.input_tokens` by model" is wrong five ways:

1. It ignores **cached tokens**, which some providers bill at a fraction (Anthropic prompt caching, OpenAI prompt caching). Mixing cached and uncached produces 5–10× cost errors on repeat traffic.
2. It attributes to the **model**, not the **feature**. Finance wants "support_agent spent $4,120 last week", not "gpt-4o-mini spent $4,120".
3. It conflates the **first-call** tokens with **tool-call** tokens. A multi-turn trace has the same prompt tokens charged repeatedly on each tool-call step; the attribution must sum by trace, not by call.
4. It uses **provider-reported** tokens only. When tokens are unavailable (older providers, local models), the tokeniser must be invoked to estimate; the row must record which path was used.
5. It mixes **cost model versions**. Pricing changes on a known date; historical rows must be re-costed against their contemporaneous pricing, not current prices.

Five load-bearing decisions anchor the solution:

1. **One `token_ledger` row per trace per model.** Primary key `(trace_id, model_pin, step_ordinal)`. A trace that made three model calls to the same model yields three rows; they are summed per trace in the view.
2. **Token types are a first-class enum**: `input_uncached`, `input_cache_read`, `input_cache_write`, `output`, `reasoning`, `tool_input`, `tool_output`. Collapsing them hides the biggest lever (cache hit rate) from the report.
3. **Cost is derived at write time against a versioned price table.** The `model_price` table is content-addressed by `(model_pin, effective_from)`. A re-cost job replays the ledger under a new price row without overwriting the original cost.
4. **Feature attribution via `trace.surface` and `trace.tool_shape`.** No new attribution plane; the schema from mod-110 ex-01 already names the feature.
5. **Reconciliation against provider invoices.** The ledger exports a daily `provider_reconciliation` report that compares `Σ ledger.cost` against the provider's invoice line items. Drift > 1% raises a ticket; > 5% pages.

```
                   span (model call)
                        │
       gen_ai.usage.input_tokens / output_tokens
       gen_ai.cache.read_tokens / write_tokens
       gen_ai.usage.reasoning_tokens
                        │
                        ▼
                  token_ledger row
         (trace, model_pin, step_ordinal, {token_type: count})
                        │
                        ▼
                  × model_price (versioned)
                        │
                        ▼
                  cost_usd per row
                        │
                        ▼
      SUM GROUP BY surface, tool_shape, tenant_class, day
                        │
                        ▼
                  feature_cost_report
```

## 2. Worked implementation

### 2.1 `tokens/schemas.py`

```python
from __future__ import annotations
from dataclasses import dataclass, field
from typing import Literal

TokenType = Literal[
    "input_uncached", "input_cache_read", "input_cache_write",
    "output", "reasoning",
    "tool_input", "tool_output",
]

Source = Literal["provider_reported", "tokenizer_estimate"]


@dataclass(frozen=True)
class TokenCounts:
    input_uncached:     int = 0
    input_cache_read:   int = 0
    input_cache_write:  int = 0
    output:             int = 0
    reasoning:          int = 0
    tool_input:         int = 0
    tool_output:        int = 0

    def total(self) -> int:
        return (self.input_uncached + self.input_cache_read + self.input_cache_write
                + self.output + self.reasoning + self.tool_input + self.tool_output)


@dataclass(frozen=True)
class ModelPrice:
    model_pin: str                      # e.g. "gpt-4o-mini-2024-07-18"
    price_version_id: str               # sha256:... content-addressed
    effective_from_ns: int
    effective_to_ns: int | None         # None = currently in force
    # USD per 1M tokens; zeros allowed where a category does not exist for the model.
    usd_per_million_input_uncached:    float
    usd_per_million_input_cache_read:  float
    usd_per_million_input_cache_write: float
    usd_per_million_output:            float
    usd_per_million_reasoning:         float = 0.0
    usd_per_million_tool_input:        float = 0.0
    usd_per_million_tool_output:       float = 0.0


@dataclass(frozen=True)
class LedgerRow:
    trace_id: str
    model_pin: str
    step_ordinal: int                   # 0-based model-call index within the trace
    span_id: str
    tokens: TokenCounts
    source: Source                      # provider_reported | tokenizer_estimate
    cost_usd: float
    price_version_id: str
    tenant_id: str
    tenant_class: str                   # paid | free | internal
    surface: str
    tool_shape: str
    ingested_ns: int
```

### 2.2 `tokens/prices.yaml` — the versioned price table

```yaml
# Prices are in USD per 1,000,000 tokens unless noted. Every row is
# content-addressed by (model_pin, effective_from_ns). A price change
# means a new row; historical traces keep their original price_version_id.
#
# The numeric values here are PLACEHOLDERS tied to the local exercise
# fixture. In a real deployment, these rows are copied from each
# provider's published rate card and reviewed on every change.
prices:
  - model_pin: "gpt-4o-mini-fixture-2024"
    price_version_id: "sha256:placeholder-mini-01"
    effective_from_ns: 1704067200000000000     # 2024-01-01T00:00:00Z
    effective_to_ns:   null
    usd_per_million_input_uncached:   0.150
    usd_per_million_input_cache_read: 0.075
    usd_per_million_input_cache_write: 0.1875
    usd_per_million_output:           0.600
  - model_pin: "gpt-4o-fixture-2024"
    price_version_id: "sha256:placeholder-4o-01"
    effective_from_ns: 1704067200000000000
    effective_to_ns:   null
    usd_per_million_input_uncached:   5.00
    usd_per_million_input_cache_read: 2.50
    usd_per_million_output:           15.00
  - model_pin: "claude-haiku-3-5-fixture-2024"
    price_version_id: "sha256:placeholder-haiku-01"
    effective_from_ns: 1704067200000000000
    effective_to_ns:   null
    usd_per_million_input_uncached:    0.80
    usd_per_million_input_cache_read:  0.08
    usd_per_million_input_cache_write: 1.00
    usd_per_million_output:            4.00
```

> `prices.yaml` is a fixture for this exercise. Real deployments read each provider's published rate card and version a row on every change. **No pricing claims about any real model are made in this file.**

### 2.3 `tokens/extract.py` — read spans into token counts

```python
from __future__ import annotations
from .schemas import Source, TokenCounts


# OpenTelemetry semantic conventions for GenAI:
# https://opentelemetry.io/docs/specs/semconv/gen-ai/
OTEL_INPUT   = "gen_ai.usage.input_tokens"
OTEL_OUTPUT  = "gen_ai.usage.output_tokens"
OTEL_REASON  = "gen_ai.usage.reasoning_tokens"

# Cache fields follow the per-vendor extensions until the cached-token
# semantic convention stabilises. We check both the common vendor name
# and the (pending) `gen_ai.usage.*_cache_tokens` naming.
CACHE_READ_KEYS  = ("gen_ai.usage.cache_read_input_tokens", "gen_ai.usage.input_cache_read_tokens")
CACHE_WRITE_KEYS = ("gen_ai.usage.cache_creation_input_tokens", "gen_ai.usage.input_cache_write_tokens")


def _first(attrs: dict, keys: tuple[str, ...], default: int = 0) -> int:
    for k in keys:
        v = attrs.get(k)
        if v is not None:
            return int(v)
    return default


def extract_from_span(attrs: dict) -> tuple[TokenCounts, Source]:
    """
    Attributes are the span's `attributes` map from the OpenTelemetry trace
    envelope. Returns (counts, source).
    """
    input_total = int(attrs.get(OTEL_INPUT, 0))
    cache_read  = _first(attrs, CACHE_READ_KEYS)
    cache_write = _first(attrs, CACHE_WRITE_KEYS)
    # Provider "input_tokens" field semantics vary. Anthropic reports
    # "input_tokens" as uncached; cached read/write are separate. OpenAI
    # reports "prompt_tokens" as the total (cached + uncached); the cached
    # subset is a nested usage.prompt_tokens_details.cached_tokens.
    # Normalise to: input_uncached = provider_input - cache_read.
    input_uncached = max(0, input_total - cache_read)

    counts = TokenCounts(
        input_uncached   = input_uncached,
        input_cache_read = cache_read,
        input_cache_write= cache_write,
        output           = int(attrs.get(OTEL_OUTPUT, 0)),
        reasoning        = int(attrs.get(OTEL_REASON, 0)),
    )
    source: Source = "provider_reported" if (
        OTEL_INPUT in attrs or OTEL_OUTPUT in attrs
    ) else "tokenizer_estimate"
    return counts, source


def tokenizer_fallback(prompt_text: str, completion_text: str, encode) -> TokenCounts:
    """
    Fallback when the provider returned no usage. `encode` is a tiktoken-like
    callable returning list[int] token ids. The ledger row records
    `source=tokenizer_estimate` so finance can audit the exposure.
    """
    return TokenCounts(
        input_uncached = len(encode(prompt_text or "")),
        output         = len(encode(completion_text or "")),
    )
```

### 2.4 `tokens/cost.py` — pricing

```python
from __future__ import annotations
from .schemas import LedgerRow, ModelPrice, TokenCounts


def price_tokens(counts: TokenCounts, price: ModelPrice) -> float:
    def cost(n: int, per_million: float) -> float:
        return (n / 1_000_000.0) * per_million
    return (
        cost(counts.input_uncached,    price.usd_per_million_input_uncached) +
        cost(counts.input_cache_read,  price.usd_per_million_input_cache_read) +
        cost(counts.input_cache_write, price.usd_per_million_input_cache_write) +
        cost(counts.output,            price.usd_per_million_output) +
        cost(counts.reasoning,         price.usd_per_million_reasoning) +
        cost(counts.tool_input,        price.usd_per_million_tool_input) +
        cost(counts.tool_output,       price.usd_per_million_tool_output)
    )


def pick_price(prices: list[ModelPrice], model_pin: str, at_ns: int) -> ModelPrice:
    """Pick the price in force at time `at_ns` for the given model pin.

    Prices are stored as half-open intervals [effective_from_ns, effective_to_ns).
    """
    candidates = [p for p in prices
                  if p.model_pin == model_pin
                  and p.effective_from_ns <= at_ns
                  and (p.effective_to_ns is None or p.effective_to_ns > at_ns)]
    if not candidates:
        raise LookupError(f"no price for {model_pin} at {at_ns}")
    # Most specific (latest effective_from) wins.
    candidates.sort(key=lambda p: -p.effective_from_ns)
    return candidates[0]
```

### 2.5 `tokens/ledger.py` — write-time pipeline

```python
from __future__ import annotations
from typing import Iterable

from .cost    import pick_price, price_tokens
from .extract import extract_from_span, tokenizer_fallback
from .schemas import LedgerRow, ModelPrice, TokenCounts


def build_row(
    *, span: dict, trace: dict, step_ordinal: int,
    prices: list[ModelPrice],
    tokenizer_encode=None,
) -> LedgerRow:
    counts, source = extract_from_span(span["attributes"])
    if source == "tokenizer_estimate" and tokenizer_encode is not None:
        counts = tokenizer_fallback(
            prompt_text     = span["attributes"].get("gen_ai.prompt",      ""),
            completion_text = span["attributes"].get("gen_ai.completion",  ""),
            encode          = tokenizer_encode,
        )
    model_pin = span["attributes"].get("gen_ai.response.model") or span["attributes"].get("gen_ai.request.model")
    if not model_pin:
        raise ValueError(f"span {span['span_id']} missing gen_ai.response.model")
    price = pick_price(prices, model_pin, at_ns=span["start_time_unix_nano"])
    cost = price_tokens(counts, price)
    return LedgerRow(
        trace_id      = trace["trace_id"],
        model_pin     = model_pin,
        step_ordinal  = step_ordinal,
        span_id       = span["span_id"],
        tokens        = counts,
        source        = source,
        cost_usd      = cost,
        price_version_id = price.price_version_id,
        tenant_id     = trace["tenant_id"],
        tenant_class  = trace["tenant_class"],
        surface       = trace["surface"],
        tool_shape    = trace["tool_shape"],
        ingested_ns   = trace["ingested_ns"],
    )


def process_trace(trace: dict, spans: list[dict], prices: list[ModelPrice],
                  tokenizer_encode=None) -> list[LedgerRow]:
    """Emit one LedgerRow per model-call span, numbered by start_time ascending."""
    model_calls = sorted(
        (s for s in spans
         if s.get("attributes", {}).get("gen_ai.operation.name") in ("chat", "completions", "messages")),
        key=lambda s: s["start_time_unix_nano"],
    )
    return [
        build_row(span=s, trace=trace, step_ordinal=i, prices=prices,
                  tokenizer_encode=tokenizer_encode)
        for i, s in enumerate(model_calls)
    ]
```

### 2.6 `tokens/sql/ledger_schema.sql` + rollup views

```sql
CREATE TABLE token_ledger (
    trace_id             UUID         NOT NULL REFERENCES trace(trace_id),
    model_pin            TEXT         NOT NULL,
    step_ordinal         INT          NOT NULL,
    span_id              TEXT         NOT NULL,

    input_uncached       INT          NOT NULL,
    input_cache_read     INT          NOT NULL,
    input_cache_write    INT          NOT NULL,
    output               INT          NOT NULL,
    reasoning            INT          NOT NULL,
    tool_input           INT          NOT NULL,
    tool_output          INT          NOT NULL,

    source               TEXT         NOT NULL,      -- provider_reported | tokenizer_estimate
    cost_usd             NUMERIC(12,8) NOT NULL,
    price_version_id     TEXT         NOT NULL,

    tenant_id            TEXT         NOT NULL,
    tenant_class         TEXT         NOT NULL,
    surface              TEXT         NOT NULL,
    tool_shape           TEXT         NOT NULL,

    ingested_ns          BIGINT       NOT NULL,

    PRIMARY KEY (trace_id, model_pin, step_ordinal)
);
CREATE INDEX token_ledger_by_surface_day
    ON token_ledger (surface, (ingested_ns / (24::BIGINT*3600*1_000_000_000)));
CREATE INDEX token_ledger_by_tenant_day
    ON token_ledger (tenant_id, (ingested_ns / (24::BIGINT*3600*1_000_000_000)));


-- Daily rollup by (feature, cohort, model). One row per day per combination.
CREATE VIEW v_feature_cost_daily AS
SELECT
    (ingested_ns / (24::BIGINT*3600*1_000_000_000))::INT * 86400 AS day_unix,
    surface, tool_shape, tenant_class, model_pin,
    SUM(input_uncached)    AS input_uncached,
    SUM(input_cache_read)  AS input_cache_read,
    SUM(input_cache_write) AS input_cache_write,
    SUM(output)            AS output,
    SUM(reasoning)         AS reasoning,
    SUM(tool_input)        AS tool_input,
    SUM(tool_output)       AS tool_output,
    SUM(cost_usd)          AS cost_usd,
    COUNT(DISTINCT trace_id) AS n_traces,
    -- Cache-hit rate is a leading indicator of the next week's cost; expose it
    -- separately per model.
    CASE WHEN SUM(input_uncached + input_cache_read) = 0 THEN 0
         ELSE SUM(input_cache_read)::NUMERIC / SUM(input_uncached + input_cache_read)
    END AS cache_hit_rate
FROM token_ledger
GROUP BY day_unix, surface, tool_shape, tenant_class, model_pin;


-- Reconciliation: one row per (day, model). Compared daily against provider
-- invoice line items by the reconciliation job (§2.9).
CREATE VIEW v_provider_reconciliation AS
SELECT
    (ingested_ns / (24::BIGINT*3600*1_000_000_000))::INT * 86400 AS day_unix,
    model_pin,
    SUM(input_uncached + input_cache_read + input_cache_write) AS input_total,
    SUM(output + reasoning)                                     AS output_total,
    SUM(cost_usd)                                               AS cost_total
FROM token_ledger
GROUP BY day_unix, model_pin;
```

### 2.7 `tokens/reconcile.py` — compare ledger to provider invoice

```python
from __future__ import annotations
from dataclasses import dataclass


@dataclass(frozen=True)
class InvoiceLine:
    day_unix: int
    model_pin: str
    input_tokens:  int
    output_tokens: int
    cost_usd:      float


def reconcile(ledger_rows: list[dict], invoice: list[InvoiceLine]) -> list[dict]:
    """
    Returns one record per (day, model) comparing ledger totals to invoice totals.

    Decision rules (also in `slo/token_recon.yaml`):
      - |Δcost| / invoice.cost < 1%:   ok
      - 1% ≤ drift < 5%:               warn (open a ticket)
      - drift ≥ 5%:                    page
    """
    inv_idx = {(ln.day_unix, ln.model_pin): ln for ln in invoice}
    ledg_idx = {(r["day_unix"], r["model_pin"]): r for r in ledger_rows}
    all_keys = sorted(set(inv_idx) | set(ledg_idx))
    out = []
    for k in all_keys:
        inv = inv_idx.get(k)
        ldg = ledg_idx.get(k)
        if inv is None:
            out.append({
                "day_unix": k[0], "model_pin": k[1],
                "status": "unexpected_ledger_only",
                "ledger_cost": ldg["cost_total"] if ldg else 0.0,
                "invoice_cost": None,
            })
            continue
        if ldg is None:
            out.append({
                "day_unix": k[0], "model_pin": k[1],
                "status": "missing_from_ledger",
                "ledger_cost": None, "invoice_cost": inv.cost_usd,
            })
            continue
        drift = (ldg["cost_total"] - inv.cost_usd) / max(inv.cost_usd, 1e-9)
        status = "ok" if abs(drift) < 0.01 else ("warn" if abs(drift) < 0.05 else "page")
        out.append({
            "day_unix": k[0], "model_pin": k[1],
            "status": status, "drift": drift,
            "ledger_cost":  ldg["cost_total"],
            "invoice_cost": inv.cost_usd,
            "ledger_input_tokens":  ldg["input_total"],
            "invoice_input_tokens": inv.input_tokens,
        })
    return out
```

### 2.8 `tokens/report.py` — the per-feature report

```python
from __future__ import annotations
from typing import Iterable


def feature_cost_report(rows: list[dict]) -> dict:
    """
    `rows` is `v_feature_cost_daily` already filtered to a window.

    Returns a nested dict:
      {
        "window_days": int,
        "features": [{
            "surface": str, "tool_shape": str,
            "n_traces": int, "cost_usd": float,
            "cost_per_trace_usd": float,
            "cache_hit_rate": float,
            "by_model": [{"model_pin": str, "cost_usd": float, "share": float}],
            "by_cohort": [{"tenant_class": str, "cost_usd": float, "share": float}],
         }, ...],
        "top_lines": [top 5 (feature, model) by cost],
      }
    """
    agg: dict = {}
    for r in rows:
        key = (r["surface"], r["tool_shape"])
        a = agg.setdefault(key, {
            "surface": r["surface"], "tool_shape": r["tool_shape"],
            "cost_usd": 0.0, "n_traces": 0,
            "input_uncached": 0, "input_cache_read": 0,
            "by_model": {}, "by_cohort": {},
        })
        a["cost_usd"]       += float(r["cost_usd"])
        a["n_traces"]       += int(r["n_traces"])
        a["input_uncached"]   += int(r["input_uncached"])
        a["input_cache_read"] += int(r["input_cache_read"])
        a["by_model"].setdefault(r["model_pin"], 0.0)
        a["by_cohort"].setdefault(r["tenant_class"], 0.0)
        a["by_model"][r["model_pin"]]     += float(r["cost_usd"])
        a["by_cohort"][r["tenant_class"]] += float(r["cost_usd"])

    features = []
    for a in agg.values():
        tot = a["cost_usd"]
        by_model = [
            {"model_pin": k, "cost_usd": v, "share": v / tot if tot else 0.0}
            for k, v in sorted(a["by_model"].items(), key=lambda kv: -kv[1])
        ]
        by_cohort = [
            {"tenant_class": k, "cost_usd": v, "share": v / tot if tot else 0.0}
            for k, v in sorted(a["by_cohort"].items(), key=lambda kv: -kv[1])
        ]
        features.append({
            "surface": a["surface"], "tool_shape": a["tool_shape"],
            "cost_usd": a["cost_usd"],
            "n_traces": a["n_traces"],
            "cost_per_trace_usd": a["cost_usd"] / a["n_traces"] if a["n_traces"] else 0.0,
            "cache_hit_rate": (
                a["input_cache_read"] / max(1, a["input_uncached"] + a["input_cache_read"])
            ),
            "by_model": by_model,
            "by_cohort": by_cohort,
        })
    features.sort(key=lambda f: -f["cost_usd"])
    top_lines = []
    for f in features:
        for m in f["by_model"][:1]:
            top_lines.append({
                "surface": f["surface"], "tool_shape": f["tool_shape"],
                "model_pin": m["model_pin"], "cost_usd": m["cost_usd"],
            })
    top_lines.sort(key=lambda t: -t["cost_usd"])
    return {
        "features": features,
        "top_lines": top_lines[:5],
    }
```

### 2.9 `tests/test_token_ledger.py`

```python
import pytest

from tokens.cost      import pick_price, price_tokens
from tokens.extract   import extract_from_span, tokenizer_fallback
from tokens.ledger    import build_row, process_trace
from tokens.reconcile import InvoiceLine, reconcile
from tokens.report    import feature_cost_report
from tokens.schemas   import ModelPrice, TokenCounts


def _mini_price(effective_from=0):
    return ModelPrice(
        model_pin="gpt-4o-mini-fixture-2024",
        price_version_id="sha256:mini-v1",
        effective_from_ns=effective_from, effective_to_ns=None,
        usd_per_million_input_uncached   = 0.150,
        usd_per_million_input_cache_read = 0.075,
        usd_per_million_input_cache_write= 0.1875,
        usd_per_million_output           = 0.600,
    )


def test_extract_splits_cached_from_uncached_input():
    attrs = {
        "gen_ai.usage.input_tokens":                      1200,
        "gen_ai.usage.cache_read_input_tokens":            900,
        "gen_ai.usage.cache_creation_input_tokens":        100,
        "gen_ai.usage.output_tokens":                      300,
    }
    counts, src = extract_from_span(attrs)
    assert src == "provider_reported"
    assert counts.input_uncached   == 300           # 1200 - 900 cached
    assert counts.input_cache_read == 900
    assert counts.input_cache_write== 100
    assert counts.output           == 300


def test_extract_falls_back_to_estimator():
    attrs = {}
    _, src = extract_from_span(attrs)
    assert src == "tokenizer_estimate"


def test_tokenizer_fallback_counts_tokens():
    counts = tokenizer_fallback("hello world", "hi", encode=lambda s: s.split())
    assert counts.input_uncached == 2
    assert counts.output == 1


def test_cost_matches_price_table():
    counts = TokenCounts(input_uncached=1_000_000, input_cache_read=1_000_000, output=1_000_000)
    assert price_tokens(counts, _mini_price()) == pytest.approx(0.150 + 0.075 + 0.600, rel=1e-6)


def test_pick_price_respects_time_windows():
    p_old = ModelPrice("m", "sha256:p1", effective_from_ns=100, effective_to_ns=200,
                        usd_per_million_input_uncached=1, usd_per_million_input_cache_read=0,
                        usd_per_million_input_cache_write=0, usd_per_million_output=5)
    p_new = ModelPrice("m", "sha256:p2", effective_from_ns=200, effective_to_ns=None,
                        usd_per_million_input_uncached=2, usd_per_million_input_cache_read=0,
                        usd_per_million_input_cache_write=0, usd_per_million_output=4)
    assert pick_price([p_old, p_new], "m", 150).price_version_id == "sha256:p1"
    assert pick_price([p_old, p_new], "m", 250).price_version_id == "sha256:p2"


def test_build_row_requires_model_pin():
    trace = {
        "trace_id": "01925a5d-2b7e-7f71-9b0c-6b9a3d6a0b23",
        "tenant_id": "acme", "tenant_class": "paid",
        "surface": "support_agent", "tool_shape": "tool_call",
        "ingested_ns": 1_705_000_000_000_000_000,
    }
    span = {
        "span_id": "s1", "start_time_unix_nano": 1_704_200_000_000_000_000,
        "attributes": {
            "gen_ai.operation.name": "chat",
            "gen_ai.usage.input_tokens":  120,
            "gen_ai.usage.output_tokens": 40,
        },
    }
    with pytest.raises(ValueError):
        build_row(span=span, trace=trace, step_ordinal=0,
                  prices=[_mini_price(effective_from=1_700_000_000_000_000_000)])


def test_process_trace_emits_one_row_per_model_call_in_order():
    trace = {
        "trace_id": "t", "tenant_id": "acme", "tenant_class": "paid",
        "surface": "support_agent", "tool_shape": "tool_call",
        "ingested_ns": 1_705_000_000_000_000_000,
    }
    spans = [
        {"span_id": f"s{i}", "start_time_unix_nano": 1_705_000_000_000_000_000 + i,
         "attributes": {
             "gen_ai.operation.name": "chat",
             "gen_ai.response.model": "gpt-4o-mini-fixture-2024",
             "gen_ai.usage.input_tokens":  100,
             "gen_ai.usage.output_tokens": 50,
         }} for i in range(3)
    ]
    rows = process_trace(trace, spans, prices=[_mini_price()])
    assert [r.step_ordinal for r in rows] == [0, 1, 2]
    assert all(r.cost_usd > 0 for r in rows)


def test_reconcile_warns_above_1pct_and_pages_above_5pct():
    ledger = [{"day_unix": 1, "model_pin": "m",
                "input_total": 1_000_000, "output_total": 100_000,
                "cost_total": 10.00}]
    inv_ok   = [InvoiceLine(day_unix=1, model_pin="m", input_tokens=1_000_000, output_tokens=100_000, cost_usd=10.05)]
    inv_warn = [InvoiceLine(day_unix=1, model_pin="m", input_tokens=1_000_000, output_tokens=100_000, cost_usd=10.30)]
    inv_page = [InvoiceLine(day_unix=1, model_pin="m", input_tokens=1_000_000, output_tokens=100_000, cost_usd=11.00)]
    assert reconcile(ledger, inv_ok)[0]["status"]   == "ok"
    assert reconcile(ledger, inv_warn)[0]["status"] == "warn"
    assert reconcile(ledger, inv_page)[0]["status"] == "page"


def test_report_ranks_features_by_total_cost():
    rows = [
        {"surface": "support", "tool_shape": "rag",   "tenant_class": "paid",
         "model_pin": "gpt-4o-mini-fixture-2024",
         "cost_usd": 20.0, "n_traces": 10,
         "input_uncached": 100, "input_cache_read": 100,
         "input_cache_write": 0, "output": 100, "reasoning": 0,
         "tool_input": 0, "tool_output": 0},
        {"surface": "billing", "tool_shape": "chat_only", "tenant_class": "paid",
         "model_pin": "gpt-4o-mini-fixture-2024",
         "cost_usd": 5.0, "n_traces": 50,
         "input_uncached": 10, "input_cache_read": 0,
         "input_cache_write": 0, "output": 10, "reasoning": 0,
         "tool_input": 0, "tool_output": 0},
    ]
    rep = feature_cost_report(rows)
    assert rep["features"][0]["surface"] == "support"
    assert rep["features"][0]["cache_hit_rate"] == pytest.approx(0.5)
    assert rep["top_lines"][0]["cost_usd"] == 20.0
```

### 2.10 Worked example report (abbreviated)

Over a 7-day window:

| Feature | n_traces | Cost (USD) | Cost / trace | Cache hit | Top model |
|---|---|---|---|---|---|
| support_agent / tool_call | 142,811 | $4,120.33 | $0.0288 | 71% | gpt-4o-mini-fixture-2024 ($3,711) |
| onboarding / chat_only    |  48,201 |   $902.12 | $0.0187 |  8% | gpt-4o-fixture-2024 ($780) |
| billing / rag             |  17,088 |   $314.44 | $0.0184 | 33% | claude-haiku-3-5-fixture-2024 ($301) |

By cohort for support_agent / tool_call:

| tenant_class | Cost | Share |
|---|---|---|
| paid     | $3,004 | 72.9% |
| free     |   $980 | 23.8% |
| internal |   $136 |  3.3% |

Top cost lines across the window:

| # | Feature | Model | Cost |
|---|---|---|---|
| 1 | support_agent / tool_call | gpt-4o-mini-fixture-2024          | $3,711 |
| 2 | onboarding / chat_only    | gpt-4o-fixture-2024               | $780 |
| 3 | support_agent / tool_call | claude-haiku-3-5-fixture-2024     | $409 |
| 4 | billing / rag             | claude-haiku-3-5-fixture-2024     | $301 |
| 5 | onboarding / chat_only    | gpt-4o-mini-fixture-2024          | $122 |

Reconciliation against provider invoices (fixture): max drift 0.6% — ok.

## 3. Validation steps

1. **Unit tests**: `pytest -q tests/test_token_ledger.py` — all 9 tests pass.
2. **End-to-end fixture**: seed 1,000 traces across three models with known token counts; compute per-feature totals; totals match sum of fixture within rounding.
3. **Cache split correctness**: for provider spans that include both `input_tokens` and `cache_read_input_tokens`, the row's `input_uncached + input_cache_read == input_tokens`.
4. **Price version stability**: compute cost at timestamp T1 with price P1; bump prices at T2 (new row); re-cost historical rows — the historical rows' `price_version_id` stays P1 unless `re_cost=true` is set.
5. **Tokeniser fallback**: write a span with no `gen_ai.usage.*` attributes; the row carries `source='tokenizer_estimate'` and the count equals the fallback tokenizer's `len(encode(prompt))`.
6. **Primary key**: `(trace_id, model_pin, step_ordinal)` is unique; a replay with the same inputs is idempotent (via `INSERT ... ON CONFLICT DO UPDATE`).
7. **Rollup correctness**: `SUM(cost_usd) FROM token_ledger WHERE day = D` equals `SUM(cost_usd) FROM v_feature_cost_daily WHERE day = D`.
8. **Reconciliation**: provider invoice totals within 1% of ledger totals; a 6% injected drift flips `status` to `page`.
9. **PII**: no prompt or completion text appears on the ledger row — only counts, ids, and costs.
10. **Reproducibility**: re-run `process_trace` on the same fixture; output rows are byte-identical (fields deterministic; floats computed in the same order).

## 4. Rubric / review checklist

| # | Criterion | Pass | Fail |
|---|---|---|---|
| R1 | Token types are first-class (uncached/cache_read/cache_write/output/reasoning/tool_input/tool_output). | Schema column per type. | Single "input_tokens" column. |
| R2 | Cost derived from a versioned `model_price` table. | `price_version_id` column; a price change bumps the row, not overwrites. | Flat constant in code. |
| R3 | Attribution is to feature (surface, tool_shape), not to model only. | `surface` + `tool_shape` columns on ledger; daily rollup groups by them. | Only `model_pin` grouped. |
| R4 | Ledger PK is `(trace_id, model_pin, step_ordinal)` so multi-call traces are preserved. | Unit test proves 3 rows for 3 calls. | One row per trace (collapses calls). |
| R5 | Provider vs tokenizer source recorded per row. | `source` enum column. | Silent fallback. |
| R6 | Reconciliation against provider invoices runs daily with ok/warn/page thresholds. | `reconcile()` + thresholds yaml; unit test covers all three statuses. | No reconciliation. |
| R7 | PII boundary: no prompt/completion text on ledger rows. | Schema has counts only. | Raw prompts leaked. |
| R8 | Cache-hit-rate exposed per feature in the rollup. | Column in `v_feature_cost_daily`. | Only aggregate cost. |
| R9 | Report ranks features by total cost AND shows top cost lines. | Both tables present. | Only one. |
| R10 | Re-costing is additive (writes a new row), never overwriting. | Schema or job spec enforces append-only. | Updates `cost_usd` in place. |

A passing submission is 9+ of 10 with R1, R2, R3, R4, R6 hard-required.

## 5. Common mistakes

- **Treating all input tokens the same.** Cached reads are often priced at 10% of uncached. Lumping them into one column makes cost look wild and hides the cache-hit lever.
- **Collapsing multi-call traces.** A tool-using trace calls the model 3× with much of the prompt repeated. Summing only `trace.cost_usd` (from mod-110 ex-01) hides that structure; the ledger must carry per-call rows.
- **Hardcoded prices.** Prices change. A cost report from last March re-costed at today's prices is wrong by the delta. Use a versioned price table and pick by `effective_from_ns`.
- **No tokeniser fallback.** Some providers / local models don't report usage. Dropping those rows undercounts; re-counting with a tokeniser and tagging `source='tokenizer_estimate'` is honest.
- **No reconciliation with the invoice.** The invoice is the source of truth for finance. A ledger that doesn't reconcile loses credibility the first time finance asks.
- **Attributing by model only.** Finance wants "support_agent spends X"; attributing to `gpt-4o-mini` doesn't answer the question that triggers a budget conversation.
- **Ignoring cache-hit rate as a signal.** Cache hit rate going down is a leading indicator of cost rising; expose it in the daily rollup so an SRE sees it before the invoice does.
- **Updating historical cost in place.** When pricing moves, the original cost is a historical record. Re-cost into a new row; never overwrite.
- **Allowing float determinism drift.** Summing floats in different orders changes the last digits. Sort by PK before summing in the report path, so `test_reproducibility` doesn't flake.
- **Leaking prompts.** Tokens are counts; prompts are content. The ledger is the finance plane and must not carry content.

## 6. References

- NIST AI Risk Management Framework, <https://www.nist.gov/itl/ai-risk-management-framework> — `GOVERN 2.1` (resource accountability), `MANAGE 2.1` (ongoing monitoring) framing per-feature cost attribution as a governance artefact.
- OWASP Machine Learning Security Top 10, <https://owasp.org/www-project-machine-learning-security-top-10/> — `ML06` (AI supply chain) motivates recording `price_version_id` alongside every row; `ML04` motivates per-tenant attribution for membership/leakage inquiries.
- MITRE ATLAS, <https://atlas.mitre.org/> — `AML.T0040` (denial of ML service) and `AML.T0037` (model extraction) are detected first as unusual token spikes on a feature — the per-feature ledger is the detection substrate.
- OpenTelemetry semantic conventions — GenAI, <https://opentelemetry.io/docs/specs/semconv/gen-ai/>, including `gen_ai.usage.input_tokens`, `gen_ai.usage.output_tokens`, `gen_ai.response.model`, and the pending cached-token conventions.
- Local exercise reader: `lessons/mod-111-cost-latency-quality-tradeoff/exercises/exercise-05-per-feature-token-accounting.md`.
- Related solutions: this module's exercise-01 (the multi-objective report consumes `cost_usd` from this ledger), exercise-02 (router cost math reads `v_feature_cost_daily` filtered to router cohorts), exercise-03 (regression auxiliary `cost_ratio` reads ledger totals), `mod-110/exercise-04` (platform cost per team), `mod-102-trace-instrumentation` (the span attributes this ledger reads).
