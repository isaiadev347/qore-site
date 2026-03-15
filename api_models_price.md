# QORE Engine API — Models & Pricing

These endpoints are served by the QORE Engine at `QORE_API_URL`.
The public CLI calls them automatically when `QORE_API_URL` is set.

---

## Models

### `GET /v1/models`

List all models in the universal registry.

> Frontier metadata (`frontier_rank`, `ntu_mapping`) is used by routing and policy
> engines. For the full NTU definition and formula, see [docs/ntu.md](./ntu.md).

**Query parameters**

| Parameter       | Type    | Default | Description                                      |
|-----------------|---------|---------|--------------------------------------------------|
| `category`      | string  | —       | Filter: `llm`, `image`, `video`, `audio`, `embedding`, `workflow`, `website` |
| `provider`      | string  | —       | Filter by provider (e.g. `openai`, `google`)     |
| `search`        | string  | —       | Substring search on SKU ID and model name        |
| `supported_only`| boolean | `true`  | Only return models QORE can route to             |

**Response** — array of model objects:

```json
[
  {
    "sku_id": "openai:gpt-4o",
    "provider": "openai",
    "source": "native",
    "provider_model_id": "gpt-4o",
    "category": "llm",
    "modality": "text+vision",
    "status": "stable",
    "native_unit": "tokens",
    "native_pricing": { "in_per_million": 2.50, "out_per_million": 10.00 },
    "ntu_mapping": { "in_ntu_per_million": 0.0010, "out_ntu_per_million": 0.0032 },
    "metadata": { "context_window": 128000, "tags": ["vision", "reasoning"] },
    "supported_by_qore": true,
    "frontier_rank": 1,
    "created_at": "2025-01-01T00:00:00Z",
    "updated_at": "2026-03-01T00:00:00Z",
    "last_seen_from_source": "2026-03-14T00:00:00Z"
  }
]
```

**Error response**

```json
{ "error": { "code": "invalid_category", "message": "Unknown category 'xyz'", "details": {} } }
```

---

### `GET /v1/models/frontier`

Top models ranked by `frontier_rank` (lower = more frontier).

**Query parameters**

| Parameter  | Type    | Default | Description                        |
|------------|---------|---------|------------------------------------|
| `category` | string  | —       | Filter by category                 |
| `limit`    | integer | `10`    | Max results                        |

**Response** — same shape as `/v1/models`, ordered by `frontier_rank` ascending.

> `frontier_rank` reflects a composite policy score (risk, p95 latency, uptime,
> catch rate) — not cost alone. See [docs/ntu.md](./ntu.md) for routing context.

---

## Pricing

### `POST /v1/price/quote`

Price a single SKU for a given workload. The response includes both `ntu` (Normalized
Token Unit cost) and `price_usd` (USD = NTU × ntu_rate). For the full NTU formula and
behavior, see [docs/ntu.md](./ntu.md).

**Request body**

```json
{
  "sku_id": "openai:gpt-4o",
  "workload": {
    "in_tokens": 10000,
    "out_tokens": 2000
  }
}
```

Workload keys by category:

| Category    | Keys                                      |
|-------------|-------------------------------------------|
| `llm`       | `in_tokens`, `out_tokens`                 |
| `image`     | `images`                                  |
| `audio`     | `minutes`                                 |
| `video`     | `seconds`                                 |
| `embedding` | `tokens`                                  |
| `workflow`  | `runs`                                    |
| `website`   | `sites`                                   |

**Response**

```json
{
  "sku_id": "openai:gpt-4o",
  "provider": "openai",
  "category": "llm",
  "native_units": { "in_tokens": 10000, "out_tokens": 2000 },
  "ntu_in": 0.0100,
  "ntu_out": 0.0064,
  "ntu": 0.0164,
  "price_usd": 0.0109,
  "workload_echo": { "in_tokens": 10000, "out_tokens": 2000 },
  "notes": null
}
```

**Error response**

```json
{ "error": { "code": "sku_not_found", "message": "SKU 'xyz' not found", "details": {} } }
```

---

### `GET /v1/price/list`

List priced models for a category with reference workload pricing. Each entry includes
`ntu` (NTU cost for the reference workload) and `price_usd` (USD = NTU × ntu_rate).
See [docs/ntu.md](./ntu.md) for reference workload definitions and NTU semantics.

**Query parameters**

| Parameter  | Type    | Required | Description              |
|------------|---------|----------|--------------------------|
| `category` | string  | yes      | Category to list         |
| `provider` | string  | no       | Filter by provider       |

**Response** — array of objects with `sku_id`, `provider`, `category`, `native_unit`, `ntu`, `price_usd`.

```json
[
  {
    "sku_id": "openai:gpt-4o",
    "provider": "openai",
    "category": "llm",
    "native_unit": "tokens",
    "ntu": 0.0164,
    "price_usd": 0.0109
  }
]
```

---

### `GET /v1/price/ntu-rate`

Current NTU (Normalized Token Unit) reference rate. Used to convert NTU costs to
USD: `USD = NTU × ntu_rate`. See [docs/ntu.md](./ntu.md) for the full formula.

**Response**

```json
{
  "ntu_rate_usd": "<live value>",
  "currency": "USD",
  "updated_at": "2026-03-14T00:00:00Z"
}
```

> The rate value is returned live by the engine and is not hardcoded in the CLI or
> in this document. Use `qore price ntu-rate` to retrieve the current rate.

---

## Route

### `GET /v1/route/suggest`

Recommend the best SKU for a category and routing mode. Selection enforces policy
constraints (risk, p95 latency, uptime, catch rate) — a SKU that is cheapest in NTU
but violates any threshold is never selected. See [docs/ntu.md](./ntu.md) for full
NTU semantics and the constraint model.

**Query parameters**

| Parameter  | Type   | Required | Description                              |
|------------|--------|----------|------------------------------------------|
| `category` | string | yes      | `llm`, `image`, `video`, `audio`, `embedding`, `workflow`, `website` |
| `mode`     | string | yes      | `best` \| `cheapest` \| `low-latency`   |

**Response**

```json
{
  "sku_id": "openai:gpt-4o-mini",
  "provider": "openai",
  "category": "llm",
  "mode": "cheapest",
  "reason": "Lowest price_usd among supported LLM models",
  "price_usd": 0.00053,
  "ntu": 0.0008,
  "frontier_rank": 2,
  "metadata": {}
}
```

**Mode semantics**

| Mode          | Selection logic                                     |
|---------------|-----------------------------------------------------|
| `cheapest`    | Lowest `price_usd` among supported models           |
| `best`        | Lowest `frontier_rank` (highest quality tier)       |
| `low-latency` | Lowest latency hint from model metadata or heuristic|

---

## CLI Usage

```bash
# Set engine URL
export QORE_API_URL=https://api.getqore.dev

# Route suggest (requires engine)
qore route suggest llm --mode cheapest
qore route suggest video --mode best --json
qore route suggest audio --mode low-latency

# Route cost (works offline too)
qore route cost llm --ntu 5.0
qore route cost image --ntu 100 --json

# List all LLM models
qore models list --category llm

# Search across all providers
qore models list --search gemma --all

# Quote a model
qore price quote --sku openai:gpt-4o --in 10000 --out 2000

# List frontier LLMs with pricing
qore price frontier --json

# Competitive benchmark across frontier LLMs
qore test compete --category llm

# Full competitive benchmark, all categories
qore test compete --json
```

When `QORE_API_URL` is **not** set, `qore price` commands fall back to local
`providers.json` data and label all output `[offline]`.  The `qore models` and
`qore test` commands require a live engine and will exit with an error.
