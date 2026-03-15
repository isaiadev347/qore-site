# FAQ

**What is QORE?**
QORE is a control plane for AI runtime pricing, diagnostics, route integrity, and
NTU (Normalized Token Unit)-based economics measurement. It gives operators one key
across providers, one wallet, and full visibility into routing decisions.

---

**What is NTU?**
NTU (Normalized Token Unit) is QORE's internal unit for expressing AI compute
economics across incompatible provider pricing schemes — tokens, images, seconds,
runs. It is not a currency or financial instrument. See [ntu.md](./ntu.md).

---

**Is NTU a token or cryptocurrency?**
No. NTU (Normalized Token Unit) is purely an internal measurement unit used inside
QORE software. It is not a token, currency, cryptocurrency, or asset of any kind.
QORE is a software and services product, not a financial product.

---

**Why does universal API access matter?**
Managing separate API keys, separate billing dashboards, and separate rate limits for
every AI provider is operational overhead. QORE gives you one key across providers —
one place to fund, monitor, and control your AI spend.

---

**Why does route integrity matter?**
AI providers can silently substitute a cheaper or different model than the one
requested. `qore route evaluate` surfaces these substitutions: it compares
requested-vs-actual provider and model, detects cost and latency anomalies, and
classifies findings by severity — from `no_issue` to `premium_integrity_violation`.

---

**What does `qore proxy` do?**
`qore proxy` is a commercial engine command that routes live AI API calls through QORE
with a per-call fee applied. It is not implemented in this public repository. A QORE
Engine license is required. Contact [hello@getqore.dev](mailto:hello@getqore.dev).

---

**What SKU categories does QORE support?**
Seven: `llm`, `image`, `video`, `audio`, `embedding`, `workflow`, `website`.
Run `qore price list <category>` to see all SKUs and NTU rates.

---

**How do I get engine access?**
Contact [hello@getqore.dev](mailto:hello@getqore.dev).

---

**What is the refund policy?**
All purchases are final. Refunds for duplicate charges, billing errors, unauthorized
transactions, or where required by law. See [license.md](./license.md).
