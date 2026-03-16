# QORE

QORE is a control plane for AI runtime pricing, diagnostics, route integrity, and
NTU (Normalized Token Unit)-based economics measurement.

**QORE's moat:**
- **One key across providers** — a single API key gives you access to every supported
  AI provider through QORE's unified routing layer.
- **One wallet, one control surface** — a single economics layer across providers,
  models, and media types.
- **Route integrity** — full visibility into requested-vs-actual provider and model,
  with drift detection, severity classification, and confidence scoring.
- **Transparent, verifiable AI runtime economics** — every cost expressed in
  NTU (Normalized Token Unit), every decision surface exposed through the public CLI.

> This repository contains public-facing materials and the public CLI for QORE.
> It does not publish protected formulas, routing logic, or sensitive runtime internals.

---

## Why QORE Exists

AI infrastructure has a transparency problem:

- **Black-box pricing** — API costs differ across providers with no shared basis for
  comparison. Hidden markups and pricing model changes go unnoticed.
- **Fragmented keys and billing** — separate API keys, separate billing dashboards,
  and separate rate limits for every provider you use.
- **Silent model and provider switching** — your request says GPT-4o; the response
  came from something else. You have no way to know.
- **No routing transparency** — which provider actually served this call? Was it the
  one you specified? What changed and why?
- **No unified trust surface** — no single place to inspect cost, latency, uptime,
  catch rate, and route integrity across your full provider mix.

---

## Core Ideas

- **Universal API access** — one key across all supported providers.
- **One control surface** — pricing, diagnostics, wallet, route integrity, and policy
  simulation through a single CLI.
- **One wallet, one economics layer** — fund once, spend across providers.
- **Route integrity** — inspect requested-vs-actual provider and model on every call.
- **Diagnostics and trust surfaces** — self-check commands expose system health,
  NTU rate consistency, and Stripe readiness.
- **NTU (Normalized Token Unit)** — QORE's internal unit for measuring AI compute
  economics across providers, models, and media types. See [docs/ntu.md](./docs/ntu.md).

---

## What QORE Is Not

- Not a token
- Not a cryptocurrency
- Not a currency or financial instrument
- Not a speculative asset
- Not a generic AI wrapper
- Not a hosted model marketplace

---

## NTU: Normalized Token Unit

QORE introduces **NTU (Normalized Token Unit)**, a vendor-neutral compute-value metric
that normalizes cost and work across black-box AI models and modalities — not just
tokens or requests. Every LLM call, image generation, audio transcription, video
render, embedding lookup, workflow execution, and web crawl is expressed in the same
unit, enabling a single budget and routing policy across your entire AI stack.

The symbolic formula is `NTU_s(workload) = α_modality × work_units × c_s` and
`USD(workload) = NTU(workload) × ntu_rate`. Exact formula constants are internal to
the QORE Engine and are not published in this repository.

NTU is not a token, currency, cryptocurrency, or financial asset of any kind.

→ [Full NTU definition](./docs/ntu.md)

---

## Public Repository Scope

This repository contains the public CLI for QORE and associated public documentation.
It is safe to inspect and use under the MIT License. Protected formulas, routing
algorithms, validator thresholds, and sensitive runtime internals are not published here.

---

## What Customers Purchase

QORE provides software and services related to AI runtime control, diagnostics,
NTU-based economics measurement, and other supported capabilities. Some capabilities
may be available via direct licensing or integration agreements.

NTU is an internal measurement unit — it is not a token, currency, or asset of any kind.

For licensing or access inquiries: [hello@getqore.dev](mailto:hello@getqore.dev)

---

## Support / Contact

[hello@getqore.dev](mailto:hello@getqore.dev)

---

## Refund / Dispute / Cancellation Policy

**Refund Policy**
All purchases are final. Refunds are only provided for duplicate charges, verified
billing errors, unauthorized transactions, or where required by law. To request a
review: [hello@getqore.dev](mailto:hello@getqore.dev)

**Dispute Resolution**
Please contact [hello@getqore.dev](mailto:hello@getqore.dev) before initiating a
payment dispute so the issue can be reviewed directly.

**Cancellation Policy**
QORE does not currently offer recurring subscriptions unless explicitly stated
otherwise. If subscription services are introduced later, cancellation terms will
be posted here.

---

## Production Status

| Component | Status |
|-----------|--------|
| **Wallet persistence** | SQLite (WAL, thread-safe). Set `WALLET_BACKEND=sqlite` and `WALLET_SQLITE_PATH=./data/wallet.db`. |
| **Live provider calls** | OpenAI and Anthropic via raw HTTP (no SDK required). Set `OPENAI_API_KEY` / `ANTHROPIC_API_KEY`. |
| **Ledger entries** | Every provider call is recorded with NTU amount, USD amount, tokens, provider, model, and chain hash. |
| **API server** | `uvicorn server.app:app --host 0.0.0.0 --port 8000`. No Stripe required at startup. |
| **CLI proxy** | `qore proxy --prompt "..."` executes a real call, deducts NTU, writes ledger entry. |

```bash
# Start with SQLite persistence
WALLET_BACKEND=sqlite uvicorn server.app:app --port 8000

# Execute a real provider call via CLI
OPENAI_API_KEY=sk-... qore proxy --prompt "Hello" --provider openai --model gpt-4o-mini
```

---

## Install

```bash
pip install -e .
qore --help
```

Requires Python >= 3.11. Zero mandatory runtime dependencies — stdlib only.

---

## Quickstart

```bash
# NTU reference rate
qore price ntu-rate

# Quote a model
qore price quote --sku gpt-4o --in 10000 --out 2000
# NTU:   0.0100 in / 0.0032 out = 0.0132 total
# Price: risk-adjusted NTU cost at current reference rate

# Route integrity check
qore route evaluate \
  --requested-provider openai --requested-model gpt-4o \
  --actual-provider openai   --actual-model gpt-3.5-turbo \
  --expected-cost 0.051 --observed-cost 0.008
# Severity:   high_confidence_silent_substitution
# Confidence: 0.92

# Ledger snapshot
qore ledger snapshot

# Self-check
qore diag run
```

---

## Commands

| Command | Description |
|---------|-------------|
| **Status** | |
| `qore status` | Engine metrics, validator stats, wallet summary |
| `qore status validator` | Validator health and decision counters |
| **Pricing** | |
| `qore price ntu-rate` | Current NTU reference rate |
| `qore price quote --sku <id> --in <N> --out <N>` | Quote a single SKU |
| `qore price batch --file <path>` | Batch price from CSV/JSON |
| `qore price list [category]` | Full pricing table — llm / image / video / audio / embedding / workflow / website |
| `qore price frontier` | Frontier model pricing overview |
| **Ledger** | |
| `qore ledger snapshot` | Dashboard-style ledger summary |
| `qore ledger history [--limit N]` | Recent ledger events |
| **Wallet** | |
| `qore wallet balance --user <id>` | Current NTU balance |
| `qore wallet history --user <id>` | Recent wallet events |
| `qore wallet fund --amount <USD>` | Add funds (direct ledger credit; Stripe when configured) |
| `qore wallet topup --amount <USD>` | Trigger Stripe checkout from CLI |
| `qore wallet limits` | Spend limits |
| `qore wallet ntu-usage [--window-days N]` | NTU consumption over rolling window |
| `qore wallet spend-report [--window-days N]` | NTU burn rate, projected runway, cost analytics |
| **Auth** | |
| `qore auth set <key>` | Store API credentials (session + config file) |
| `qore auth show` | Display masked key and auth configuration |
| `qore auth clear` | Remove all stored credentials |
| **Diagnostics** | |
| `qore diag run` | Self-check: catalog, env, Stripe readiness, NTU consistency |
| `qore diag ntu-consistency` | Verify NTU rate consistency across catalog |
| **Policy** | |
| `qore policy simulate --category <cat> --policy-file <path>` | Simulate NTU policy constraints against catalog |
| `qore policy check --file <path>` | Validate a policy file before applying it |
| **Route Integrity** | |
| `qore route evaluate --requested-provider P --requested-model M --actual-provider P --actual-model M` | Evaluate route integrity: drift, severity, confidence |
| `qore route benchmark --provider P --model M --runs N` | Latency p50/p95/p99, catch rate, NTU efficiency score |
| **Provider** | |
| `qore provider trust-score` | Trust scores per provider — uptime, drift history, substitution incidents |
| **NTU** | |
| `qore ntu rate` | Current NTU reference rate with pricing version |
| `qore ntu explain` | Human-readable NTU explanation |
| `qore ntu drift [--window-days N]` | NTU cost drift over time per provider |
| **Transparency** | |
| `qore explain --sku <id>` | Plain English: why a SKU was routed, frontier rank, what would change it |
| `qore compare --sku-a <id> --sku-b <id> --in <N> --out <N>` | Side-by-side NTU and USD cost comparison |
| **Engine Commands** ¹ | |
| `qore proxy` | Route AI calls through QORE (engine license required) |
| `qore closed` | CLOSED loop adaptive threshold control (engine license required) |
| `qore dev` | Dev/debug surface (fund, reset-wallet, ledger dump) |

¹ `qore proxy` and `qore closed` require the private QORE Engine license.
Contact [hello@getqore.dev](mailto:hello@getqore.dev).

---

## Route Integrity Severity Levels

| Severity | Meaning |
|----------|---------|
| `no_issue` | Route matched exactly — no drift |
| `low_confidence_drift` | Route changed within allowed fallback policy |
| `medium_confidence_route_drift` | Unexpected route change, limited evidence |
| `high_confidence_silent_substitution` | Route changed with corroborating cost/latency evidence |
| `premium_integrity_violation` | Route changed in no-drift / premium mode |

---

## License

MIT — see [LICENSE](./LICENSE).

Engine capabilities: Proprietary — [hello@getqore.dev](mailto:hello@getqore.dev)

---

QORE is built and maintained by a solo developer. Response times on issues, feature
requests, and bug fixes may be slower than larger projects. If something is broken or
blocking you, email [hello@getqore.dev](mailto:hello@getqore.dev) directly — that is
the fastest path to a resolution.
