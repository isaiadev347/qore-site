# NTU — Normalized Token Unit

## How NTU Maps to Dollars

NTU is QORE's internal unit for measuring AI compute economics. Every provider call —
LLM completion, image generation, audio transcription, embedding, or workflow execution —
is expressed in NTU, giving you a single budget and control surface across your entire
AI stack.

**The conversion is simple:**

```
USD_cost = NTU_cost × ntu_rate
NTU_cost = USD_cost / ntu_rate
```

The default `ntu_rate` is `0.001 USD / NTU` (i.e. 1 NTU = $0.001 USD).

**Example:**

| Call | USD cost | NTU cost (rate=0.001) |
|------|----------|----------------------|
| GPT-4o-mini, 1K in / 500 out | $0.000195 | 0.195 NTU |
| Claude 3.5 Haiku, 1K in / 500 out | $0.000125 | 0.125 NTU |
| GPT-4o, 1K in / 500 out | $0.007500 | 7.500 NTU |

The `ntu_rate` can change over time as market rates shift. QORE tracks this via the
`RatePublisher` and exposes the current rate at `GET /ntu/rate` and via `qore price ntu-rate`.

---

## NTU and the QORE Wallet

When you fund your wallet, USD is converted to NTU:

```
NTU_credited = (USD_funded / ntu_rate) × (1 - spread_pct)
```

When you make a provider call, NTU is deducted:

```
NTU_deducted = USD_cost / ntu_rate
```

Your wallet balance is always expressed in NTU. You can see the current balance via:

```bash
qore wallet balance
```

And convert back to USD at any time using the current rate:

```bash
qore price ntu-rate   # shows current rate
```

---

## NTU in the Ledger

Every provider call writes a ledger entry with:

| Field | Description |
|-------|-------------|
| `amount_ntu` | NTU deducted (negative value = debit) |
| `amount_usd` | USD equivalent at call time |
| `ntu_rate` | Rate used at time of call |
| `provider` | Provider that served the call |
| `model` | Model used |
| `input_tokens` / `output_tokens` | Token counts |
| `pricing_version_id` | Pricing catalog version |

Retrieve via:

```bash
qore ledger snapshot
# or
GET /wallet/ledger?wallet_id=default
```

---

## What NTU Is Not

NTU is not a token, currency, cryptocurrency, stablecoin, or financial instrument of
any kind. It is a software-internal unit of measurement — the same way "kilobytes" or
"requests-per-second" are units without being currencies.

NTU values are not transferable, tradeable, or redeemable outside QORE's software system.
