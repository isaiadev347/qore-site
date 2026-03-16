# QORE API Reference

The production API server is `server/app.py`. It runs on port 8000 by default,
requires no Stripe at startup, and persists all ledger entries to SQLite when
`WALLET_BACKEND=sqlite` is set.

> Legacy server: `server/api_server.py` (uses in-memory ledger, Stripe optional).

---

## Start the server

```bash
# Production (SQLite persistence, live provider calls)
WALLET_BACKEND=sqlite OPENAI_API_KEY=sk-... uvicorn server.app:app --host 0.0.0.0 --port 8000

# Development (in-memory)
uvicorn server.app:app --host 0.0.0.0 --port 8000
```

Environment variables:

| Variable | Default | Description |
|----------|---------|-------------|
| `WALLET_BACKEND` | `memory` | `memory` or `sqlite` |
| `WALLET_SQLITE_PATH` | `./data/wallet.db` | SQLite DB path |
| `OPENAI_API_KEY` | — | Enables OpenAI provider |
| `ANTHROPIC_API_KEY` | — | Enables Anthropic provider |

---

## Endpoints

## Wallet & Ledger Endpoints

### `GET /wallet/balance`

Returns the NTU and estimated USD balance for a wallet.

**Query parameters:** `wallet_id` (default: `default`)

**Response:**
```json
{
  "wallet_id": "default",
  "balance_ntu": 9850.25,
  "balance_usd_approx": 9.85025,
  "ntu_rate": 0.001,
  "event_count": 12
}
```

---

### `GET /wallet/history`

Returns recent wallet events for a wallet.

**Query parameters:** `wallet_id` (default: `default`), `limit` (default: `50`)

**Response:**
```json
{
  "wallet_id": "default",
  "events": [
    {
      "event_type": "debit",
      "ntu_delta": -0.195,
      "ntu_balance_after": 9850.25,
      "usd_paid": 0.0,
      "provider": "openai",
      "model": "gpt-4o-mini",
      "ts": 1742000000.0
    }
  ]
}
```

---

### `GET /wallet/ledger`

Returns provider call ledger entries for a wallet. Each entry is one executed
and priced provider API call.

**Query parameters:** `wallet_id` (default: `default`), `limit` (default: `50`)

**Response:**
```json
{
  "wallet_id": "default",
  "entries": [
    {
      "id": 1,
      "wallet_id": "default",
      "provider": "openai",
      "model": "gpt-4o-mini",
      "input_tokens": 12,
      "output_tokens": 8,
      "amount_ntu": 0.195,
      "amount_usd": 0.000195,
      "ntu_rate": 0.001,
      "pricing_version_id": "2025Q1-v1",
      "source": "provider_call",
      "correlation_id": "eng-abc123",
      "ts": 1742000000.0
    }
  ],
  "count": 1
}
```

---

### `POST /proxy`

Execute a live provider call, price in NTU, and record in the ledger.

**Request:**
```json
{
  "prompt": "Hello, world",
  "provider": "openai",
  "model": "gpt-4o-mini",
  "max_tokens": 1024,
  "temperature": 0.7,
  "wallet_id": "default"
}
```

Use `messages` (OpenAI chat format) instead of `prompt` for multi-turn:

```json
{
  "messages": [{"role": "user", "content": "Hello"}],
  "provider": "anthropic",
  "model": "claude-3-5-haiku-20241022"
}
```

**Response:**
```json
{
  "request_id": "abc12345",
  "completion": "Hello! How can I help?",
  "provider_used": "openai",
  "model_used": "gpt-4o-mini",
  "usage": {"input_tokens": 10, "output_tokens": 8},
  "ntu_cost": 0.195,
  "usd_cost": 0.000195,
  "ledger_entry": {
    "event_type": "debit",
    "amount_ntu": 0.195,
    "source": "provider_call",
    "correlation_id": "eng-abc12345",
    "ts": 1742000000.0
  },
  "pricing_version_id": "2025Q1-v1",
  "ntu_rate": 0.001,
  "timestamp": "2026-03-15T12:00:00+00:00"
}
```

| Status | Condition |
|--------|-----------|
| `400` | Neither `prompt` nor `messages` provided |
| `502` | Provider call failed |
| `503` | Engine router not initialised (no API keys set) |

---

## Health / Metrics

### `GET /health`

Health check and configuration status.

**Response:**
```json
{
  "status": "ok",
  "stripe_enabled": false,
  "fees_active": false,
  "spread_pct": 0.0,
  "consumption_fee_pct": 0.0,
  "ntu_rate": {
    "rate": 0.001,
    "pricing_version_id": "2025Q1-v1"
  },
  "timestamp": "2026-03-12T03:05:00Z"
}
```

---

### `POST /wallet/fund`

Fund a user wallet from a USD amount.

**Request:**
```json
{
  "user_id": "alice",
  "usd_amount": 50.00,
  "correlation_id": "order-123"
}
```

**Response:**
```json
{
  "user_id": "alice",
  "gross_ntu": 50000.0,
  "spread_fee_ntu": 0.0,
  "net_ntu": 50000.0,
  "new_balance": 50000.0,
  "correlation_id": "order-123"
}
```

**Notes:**
- `spread_fee_ntu` depends on `SPREAD_PCT` env var (0.0 = no fee)
- Idempotency is enforced via `correlation_id`

---

### `GET /ntu/rate`

Current NTU reference rate.

**Response:**
```json
{
  "rate": 0.001,
  "unit": "NTU",
  "currency": "USD",
  "pricing_version_id": "2025Q1-v1",
  "effective_at": "2025-01-01T00:00:00Z",
  "timestamp": "2026-03-12T03:05:00Z"
}
```

---

### `POST /usage/simulate`

Compute hypothetical NTU cost for a workload (read-only, no ledger mutation).

**Request:**
```json
{
  "provider": "openai",
  "model": "gpt-4o",
  "input_tokens": 1000,
  "output_tokens": 500
}
```

**Response:**
```json
{
  "provider": "openai",
  "model": "gpt-4o",
  "input_tokens": 1000,
  "output_tokens": 500,
  "usd_cost": 0.0075,
  "ntu_cost": 7.5
}
```

---

### `POST /stripe/webhook`

Stripe webhook receiver. Requires `STRIPE_ENABLED=true` and `STRIPE_WEBHOOK_SECRET` to be set.

Handles `checkout.session.completed` events to credit user wallets after payment.

**Request:** Stripe signed webhook payload (see Stripe docs)

**Response:**
```json
{"status": "ok"}
```

---

### `POST /chat` — [PRIVATE ENGINE]

Execute a routed chat call. Delegates to private QORE engine.

**Request:**
```json
{
  "messages": [{"role": "user", "content": "Hello"}],
  "provider": "openai",
  "model": "gpt-4o",
  "max_tokens": 1024
}
```

**Response:** Structured result including completion, usage, routing decision.

---

### `POST /route` — [PRIVATE ENGINE]

Get a routing decision without executing the call.

---

### `POST /oracle` — [PRIVATE ENGINE]

Get pricing across all providers for a given token count.

---

### `GET /metrics` — [PRIVATE ENGINE]

System-level metrics from the routing engine.
