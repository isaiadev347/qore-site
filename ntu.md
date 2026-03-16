# NTU (Normalized Token Unit)

**NTU (Normalized Token Unit)** is QORE's internal unit for measuring the economics
of AI compute across providers, modalities, and pricing regimes.

NTU is not a public token, currency, or financial instrument. It exists only inside
QORE infrastructure as a measurement, pricing, and routing layer.

> **Disclaimer:** Example values in this document are illustrative only and do not
> reflect internal calibration constants or proprietary parameters.

---

## Why NTU Exists

AI providers price compute in incompatible units. There is no shared basis for
comparison across modalities or providers:

| Category         | Native unit                           |
|------------------|---------------------------------------|
| LLM APIs         | tokens per million (in/out separate)  |
| Image generation | per image                             |
| Video generation | per second of output                  |
| Audio — STT/TTS  | per minute                            |
| Embeddings       | per million tokens                    |
| Workflow runs    | per execution                         |
| Web scraping     | per site crawled                      |

A budget expressed in GPT-4o tokens means nothing when switching to Runway video or
ElevenLabs audio. A routing policy expressed in per-image cost cannot be applied to
an LLM fallback.

NTU solves this by providing a single provider-neutral unit that maps all of the
above into one economics layer. Policy, budgeting, routing decisions, and route
integrity checks can then be expressed in NTU terms — independent of any specific
provider's pricing model or native unit.

---

## NTU Formula (Symbolic)

The NTU for a SKU `s` and a given workload is:

```
NTU_s(workload) = α_modality × work_units × c_s
```

The USD cost is:

```
USD(workload) = NTU(workload) × ntu_rate
```

Where:
- `α_modality` — modality normalization coefficient (internal to QORE Engine; not published)
- `work_units` — raw units consumed (tokens, images, seconds, minutes, runs, sites)
- `c_s` — per-SKU calibration factor; accounts for provider-specific risk, quality, and
  market conditions (internal to QORE Engine; not published)
- `ntu_rate` — current NTU reference rate in USD/NTU, returned live by `qore price ntu-rate`

For LLM workloads with separate input/output pricing:

```
NTU_s(workload) = (α_in × in_tokens + α_out × out_tokens) × c_s
```

> The exact formula constants, normalization weights, and provider-specific
> parameters are proprietary and are not published in this repository.

---

## Worked Example

> **Numbers below are illustrative only.** They show the structure of the
> computation; actual rates differ and are returned live by the engine.

**Scenario:** A 55,000-token LLM call (50,000 input + 5,000 output) through a
budget frontier model.

**Step 1 — Compute NTU using the symbolic formula:**

```
NTU_s(workload) = (α_in × 50,000 + α_out × 5,000) × c_s
               ≈ k_llm × (in_ntu_per_million × 50,000 + out_ntu_per_million × 5,000)
                         / 1,000,000
```

Where `k_llm` rolls up `α_modality` and `c_s` for this SKU. Using illustrative
per-million rates:

| Field                   | Illustrative value |
|-------------------------|--------------------|
| in_tokens               | 50,000             |
| out_tokens              | 5,000              |
| in_ntu_per_million      | 0.30 (illustrative)|
| out_ntu_per_million     | 0.91 (illustrative)|
| NTU (in)                | 0.0150             |
| NTU (out)               | 0.0046             |
| **NTU (total)**         | **0.0196**         |

**Step 2 — Convert NTU → USD:**

```
USD(workload) = NTU(workload) × ntu_rate
             = 0.0196 × [live rate from qore price ntu-rate]
             ≈ $0.013 USD  (illustrative)
```

The provider's own USD cost for this workload may differ; NTU normalizes the
economics so the same budget math applies regardless of provider or modality.

---

## Raw Tokens Fail — NTU Stable

Using raw token counts (or raw native units) as a cross-provider metric fails in
several common cases. NTU is designed to remain stable where raw metrics break down.

**Same 55k-token workload (50k in / 5k out), two SKUs from the catalog:**

| SKU                 | Raw tokens | Illustrative NTU | Why they differ             |
|---------------------|------------|------------------|-----------------------------|
| `gemini-2.0-flash`  | 55,000     | ~0.020           | Budget tier, low c_s        |
| `claude-3-opus`     | 55,000     | ~1.700           | Premium tier, high c_s      |

Same prompt, same token count — 85× difference in NTU cost. Raw token count alone
tells you nothing about pricing tier, quality class, or expected economics.

**General failure modes:**

| Scenario                              | Raw tokens     | NTU           |
|---------------------------------------|----------------|---------------|
| Same model, different providers       | Identical      | Differs (risk, uptime, c_s) |
| Different models, same token count    | Identical      | Differs (quality tier, calibration) |
| LLM vs image (equivalent workload)    | Not comparable | Comparable    |
| LLM vs audio (equivalent workload)    | Not comparable | Comparable    |
| Provider silently upgrades model      | May be lower   | NTU reflects real cost shift |
| Provider silently downgrades model    | May be lower   | NTU detects cost anomaly |
| Bulk pricing discount                 | Unchanged      | NTU reflects effective cost |

---

## Cross-Modality Workflow Example

The following illustrates a single composite workflow that mixes modalities:
one LLM call + one image generation + one minute of audio transcription.
Numbers are illustrative; live rates from `qore price ntu-rate` and
`qore price list <category>`.

| Step                   | Category  | Work units              | Illust. NTU | Illust. USD |
|------------------------|-----------|-------------------------|-------------|-------------|
| LLM call (10k in/2k out) | llm     | 10,000 in / 2,000 out   | 0.013       | $0.009      |
| Image generation (1)   | image     | 1 image                 | 0.008       | $0.005      |
| Audio transcription (1 min) | audio | 1 minute               | 0.006       | $0.004      |
| **Totals**             |           |                         | **0.027**   | **$0.018**  |

Because all three components are expressed in NTU, a single budget policy applies
across the entire workflow — no per-modality accounting required.

**Single-modality NTU equivalence (illustrative, 1.0 NTU):**

| Category   | Native unit    | Approx. quantity for 1 NTU |
|------------|----------------|----------------------------|
| llm        | tokens         | ~75,000 in / 15,000 out (frontier) |
| image      | images         | ~125 images (standard)     |
| audio      | minutes        | ~167 minutes               |
| video      | seconds        | ~7 seconds                 |
| embedding  | million tokens | ~15,000 million tokens     |
| workflow   | runs           | ~200 runs                  |
| website    | sites          | ~20 sites                  |

> Live values returned by `qore price list <category>` and `qore price quote`.

---

## How NTU Is Used in QORE

Every SKU in the QORE catalog has an NTU rate. The public CLI exposes these:

```bash
# Current NTU reference rate
qore price ntu-rate

# Quote a model
qore price quote --sku gpt-4o --in 10000 --out 2000

# List pricing for a category
qore price list llm
qore price list image
qore price list video

# Route suggest — pick best SKU for category and mode
qore route suggest llm --mode cheapest
qore route suggest video --mode best

# Cost estimate for a NTU budget
qore route cost llm --ntu 1.0
```

NTU is used across:

- **Pricing** — every SKU quote returns NTU cost and USD price
- **Route suggestion** — `qore route suggest` selects SKUs by NTU efficiency and policy
- **Route cost** — `qore route cost` estimates USD spend for a given NTU budget
- **Policy simulation** — routing policies are defined in NTU constraints
  (`qore policy simulate`)
- **Budget governance** — wallet and NTU usage tracked together
  (`qore wallet ntu-usage`)
- **Route integrity** — cost and latency deltas expressed in NTU to detect
  silent provider substitutions (`qore route evaluate`)

---

## Framework vs Candidate Standard

NTU is currently a **QORE-internal framework**: a proprietary normalization layer
used within QORE's pricing, routing, and governance surfaces.

The long-term goal is for NTU — or a derivative — to become a **candidate standard**
for expressing AI compute economics across providers.

| Dimension              | Framework (now)                        | Candidate standard (future)             |
|------------------------|----------------------------------------|-----------------------------------------|
| Scope                  | Internal to QORE infrastructure        | Cross-operator, cross-provider          |
| Formula visibility     | Proprietary                            | Auditable reference formula             |
| Adoption requirement   | QORE integration only                  | Third-party adoption / interop          |
| Governance             | QORE engineering                       | Open governance body                    |
| Audit surface          | Internal only                          | Public audit trail                      |

QORE is not currently pursuing external standardization. This section describes
the design intent, not a commitment or roadmap item.

---

## Attack-Pattern Mitigations

NTU is designed to resist several classes of measurement gaming. Each named pattern
below has a corresponding mitigation built into the engine or validator layer.

**1. Proxy gaming.** Routing calls through an intermediary that reports a cheaper
SKU to the accounting layer while forwarding to a costlier one. Mitigation: route
integrity (`qore route evaluate`) validates requested-vs-actual provider and model
on every call and classifies drift by severity and confidence.

**2. Boundary manipulation.** Structuring workloads to land just below pricing
tier boundaries (e.g., 999 tokens instead of 1,001) to systematically exploit
threshold effects. Mitigation: NTU is continuous, not bucketed; no sharp threshold
exists for a caller to target.

**3. Denominator games.** Inflating the denominator in a per-unit rate (e.g.,
reporting more "tokens" than were consumed) to make cost-per-output appear lower.
Mitigation: NTU is computed from the actual workload returned by the provider
API, not from caller-reported units.

**4. Task cherry-picking.** Benchmarking only on workloads where a cheap SKU
performs well to make it appear equivalent to a premium SKU. Mitigation: `qore
test compete` evaluates all frontier SKUs against the same reference workload;
results include `catch_rate` and policy-compliance fields.

**5. Benchmark overfitting.** Tuning a model or routing policy specifically to
score well on the public benchmark workload rather than real workloads. Mitigation:
reference workloads in `providers.json` are a minimum floor; frontier rank is a
composite of risk, uptime, latency, and catch rate — not benchmark score alone.

**6. Splitting / packing.** Splitting one large request into many small ones (or
packing unrelated work into a single context) to exploit per-request minimums or
bulk discounts. Mitigation: NTU is additive per work unit; aggregation rules make
total NTU invariant to split/pack strategies.

**7. Hidden quality tradeoffs.** Accepting lower output quality (shorter responses,
degraded images) to reduce token or compute consumption without the caller noticing.
Mitigation: `catch_rate` and validator acceptance rate are tracked per SKU and
surface quality degradation as a policy signal.

**8. Latency shifting.** Choosing artificially low-latency SKUs to pass latency
policy gates while delivering lower quality or reliability. Mitigation: latency
is one of several inputs to frontier rank; SKUs must also pass catch rate and
uptime thresholds before being eligible for routing.

**9. Reporting manipulation.** Returning altered metadata (token counts, duration,
image dimensions) to reduce the reported NTU cost. Mitigation: the engine validates
workload metadata against provider API responses; discrepancies trigger a validator
alert.

**10. Adversarial input shaping.** Crafting inputs specifically to trigger cheap
code paths in a provider (e.g., inputs that cache or short-circuit internally) while
billing at the full rate. Mitigation: NTU is derived from the provider's reported
consumption, not from QORE-estimated complexity.

**11. Cross-subsidy masking.** Offering artificially low prices on one SKU category
(e.g., embeddings) while marking up another (e.g., LLM) to hide the true blended
cost. Mitigation: `qore price list <category>` and `qore route cost` expose per-SKU
and per-category NTU costs independently; no cross-category blending is hidden.

**12. Strategic abstention.** A provider selectively refusing to serve certain
workload types (e.g., high-risk content) to improve its measured catch rate and
frontier rank on the remaining workloads. Mitigation: abstention is tracked as a
separate counter; frontier rank penalizes excessive rejection rates alongside catch
rate.

---

## NTU Views Are Never Optimized in Isolation

NTU cost is one signal among several. QORE routing and policy decisions always
consider NTU alongside:

- **p95 latency** — response time under load
- **uptime** — provider availability over the trailing window
- **catch rate** — validator acceptance rate for this SKU
- **risk adjustment** — provider-specific reliability and pricing risk
- **policy constraints** — operator-defined NTU budgets, category limits, and
  provider allow/blocklists

A SKU that is cheapest in NTU but violates latency or uptime thresholds will not
be selected by `qore route suggest --mode cheapest`. Routing modes optimize within
policy constraints, not against them.

---

## What NTU Counts

NTU counts the economic value of AI compute consumed. It is designed to capture:

- **Token consumption** for LLM calls (input and output separately)
- **Generative compute** for image, video, and audio production
- **Retrieval and embedding compute** for vector search workloads
- **Orchestration compute** for workflow automation runs
- **Crawl and extraction compute** for web scraping operations

NTU does **not** count:

- Wall-clock time waiting for human input
- Network transfer or infrastructure costs outside AI compute
- Storage costs
- Idempotent re-reads or cache hits (depending on engine configuration)
- Failed requests that did not consume provider compute

---

## Sampling, Aggregation, and Failure Handling

**Sampling.** NTU is computed per-call, not sampled. Every routed request
contributes to NTU totals. Diagnostic sampling (e.g., for latency histograms)
does not affect NTU accounting.

**Aggregation.** NTU totals are summed across calls within a wallet or budget
window. Aggregation is additive — there is no rounding or bucketing that could
obscure individual call costs.

**Failure handling.** Failed requests are handled as follows:

| Failure type                            | NTU charged? |
|-----------------------------------------|--------------|
| Provider error before compute begins    | No           |
| Provider timeout after partial compute  | Partial (if provider charges) |
| Request rejected by QORE validator      | No           |
| Request rejected by provider            | No           |
| Rate-limited and not retried            | No           |
| Successfully retried after failure      | Yes (for the successful attempt only) |

Failures are visible in `qore ledger history` and `qore wallet ntu-usage`.
NTU totals reflect actual compute consumed, not attempted compute.

---

## What Is Not Published

The following are internal to the QORE Engine and are not in this repository:

- Exact formula mapping raw workloads to NTU
- Normalization factor values and provider-specific weights
- Validator logic and internal decision thresholds
- Routing algorithms and provider selection weights
- Provider-specific cost adjustments and risk factors
- Causal graph internals and risk scoring logic

---

## Disclaimer

This page is purely informational. It describes a technical internal unit used inside
QORE software.

**NTU is not a token, currency, cryptocurrency, or financial asset of any kind.**
QORE is a software and services product, not a financial product. Nothing on this
page constitutes financial advice.

Patent pending.
