# Pricing

DOS AI uses a simple **pay-as-you-go** pricing model. You only pay for the tokens you use, with no minimum commitments, no monthly fees, and no hidden charges.

## Zero-Markup Policy

DOS AI operates on a strict **Zero-Markup Policy**:
- **100% Provider Published List Price**: We sell cloud and third-party models at exactly 100% of the upstream provider's official published list price (`markup_pct = 0`).
- **No Hidden Fees or Surcharges**: You never pay more for tokens than you would by calling providers directly.
- **Single Transparent Bill**: Consolidate multiple model providers into one unified account, API key, and balance without paying markup penalties.

---

## Free Tier

Every new account receives **$5.00 in free credits** to get started. This is enough for substantial experimentation and prototyping before you need to add funds.

| Model | Approximate Free Usage |
| :---- | :--------------------- |
| **GPT-6 Luna** | ~50 million input tokens (or 10M output tokens) |
| **DOS Curated Endpoint (`dos`)** | ~71 million input tokens (or 10M output tokens) |
| **DeepSeek V4.1 Flash** | ~16.6 million input tokens (or 4.1M output tokens) |
| **Gemini 3.8 Flash** | ~6.3 million tokens |

> Free credits do not expire. No credit card is required to start.

---

## Per-Token Pricing

Pricing is calculated per **1 million tokens** (input, output, and cached input). All rates are statically database-driven from the Supabase catalog (`dosai.model_pricing`) and strictly follow official provider published prices.

### Smart Router & Curated Platform Models

| Model | Provider | Input Price (per 1M) | Output Price (per 1M) | Cached Input (per 1M) | Billing Policy |
| :---- | :------- | :------------------- | :-------------------- | :-------------------- | :------------- |
| **Smart Router (`auto`)** | Dynamic | Variable | Variable | Variable | Billed at target model's exact rate |
| **DOS Curated Endpoint (`dos`)** | DOS.AI (Singapore cluster) | **$0.07** | **$0.50** | — | Wholesale fixed rate (Singapore GPU default) |

*Note: Calls targeting `model: "dos"` execute on the Singapore GPU cluster at the fixed $0.07/$0.50 rate. Direct calls to individual curated engines (e.g. `gpt-6-luna`, `gemini-3.8-flash`, `deepseek-v4.1-flash`) or auto-escalations via `auto` are billed at each target engine's exact published rate.*

### Frontier & Partner Cloud Models

| Model | Provider | Input Price (per 1M) | Output Price (per 1M) | Cached Input (per 1M) | Context Window |
| :---- | :------- | :------------------- | :-------------------- | :-------------------- | :------------- |
| **GPT-6 Luna** | OpenAI / Azure | **$0.10** | **$0.50** | $0.01 | 1.05M |
| **GPT-6 Sol** | OpenAI / Azure | $2.00 | $10.00 | $0.20 | 1.05M |
| **GPT-6 Astra** | OpenAI | $5.00 | $25.00 | $0.50 | 1.05M |
| **Gemini 3.8 Flash** | Google Cloud | $0.79 | $3.94 | $0.20 | 1M |
| **DeepSeek V4.1 Flash** | Alibaba / DeepSeek | $0.30 | $1.20 | $0.03 | 1M |
| **Qwen 3.8 Flash** | Alibaba Cloud | $0.15 | $0.47 | $0.016 | 1M |
| **Qwen 3.8 27B** | Alibaba Cloud | $0.50 | $3.00 | $0.10 | 1M |
| **Qwen 3.8 Max** | Alibaba Cloud | $2.00 | $6.00 | $0.25 | 1M |
| **Claude Sonnet 5** | Anthropic | $3.00 | $15.00 | $0.30 | 1M |
| **Claude Opus 5** | Anthropic | $15.00 | $75.00 | $1.50 | 1M |

> Prices are statically database-driven from the Supabase catalog and updated strictly via migrations. Check the [dashboard](https://app.dos.ai/models) or `GET /v1/models` for real-time account pricing.

### Prompt Caching Savings

For supported models (including GPT-6 series, DeepSeek V4.1, and Qwen 3.8), repeated prompt prefixes automatically benefit from prompt caching:
- **GPT-6 Luna**: Cached input tokens are billed at **$0.01 / 1M tokens** (90% savings).
- **Qwen 3.8 Flash**: Cached input tokens are billed at **$0.016 / 1M tokens** (89% savings).
- **Qwen 3.8 27B**: Cached input tokens are billed at **$0.10 / 1M tokens** (80% savings).
- **Qwen 3.8 Max**: Cached input tokens are billed at **$0.25 / 1M tokens** (87.5% savings).
- **DeepSeek V4.1 Flash**: Cached input tokens are billed at **$0.03 / 1M tokens** (90% savings).

---

### What is a Token?

A token is roughly 3-4 characters of English text, or about 0.75 words. For example:

- "Hello, world!" = approximately 4 tokens
- A typical 500-word blog post = approximately 650-700 tokens
- A full 128K context window = approximately 96,000 words
- A 1.05M context window = approximately 780,000 words

---

## How Billing Works

1. **Add credits** to your account via the [dashboard](https://app.dos.ai).
2. **Make API calls** as normal. Each request deducts tokens used from your balance.
3. **Monitor usage** in real time through the dashboard billing page.

Token usage is calculated after each request completes. Both input tokens (your prompt) and output tokens (the model's response) are counted and billed at the rates above.

### Usage Tracking

Every API response includes a `usage` object showing exactly how many tokens were consumed:

```json
{
  "usage": {
    "prompt_tokens": 125,
    "completion_tokens": 320,
    "total_tokens": 445,
    "prompt_tokens_details": {
      "cached_tokens": 100
    },
    "cost": 0.0001725
  }
}
```

You can also view historical usage and spending breakdowns on the [dashboard](https://app.dos.ai).

## Billing History & Receipts

You can view your complete transaction history at any time on the [Billing Dashboard](https://app.dos.ai/billing):

- **Comprehensive Ledger**: View all credit purchases, signup bonuses, referral rewards, refunds, and usage charges in a unified, sortable table.
- **Direct Stripe Receipts & Invoices**: For any card deposit, subscription, or automated recharge processed via Stripe, click the **Receipt / Invoice** button to instantly open official Stripe receipts and hosted invoice PDFs in a secure new tab.
- **Privacy Protection**: Referral commissions automatically redact internal referee identifiers (`[redacted]`) to safeguard team and member privacy.

---

## Enterprise & Volume Discounts

For organizations with high-volume needs, we offer custom agreements:

- **Volume discounts** for sustained usage above $100/month
- **Dedicated capacity** with guaranteed throughput
- **Custom rate limits** tailored to your workload
- **Priority support** with SLA guarantees

Contact us at **support@dos.ai** to discuss enterprise pricing.

---

## FAQ

### Do free credits expire?

No. Your free credits remain in your account until used.

### Is there a minimum top-up amount?

The minimum credit purchase is $5.00.

### What happens when my balance reaches zero?

API requests will return a `402 Payment Required` error. Add credits to resume usage immediately. No data is lost.

### Can I set spending limits?

Yes. You can configure monthly spending alerts and hard limits in the dashboard settings.

### Are there any hidden fees?

No. You pay only for the tokens you consume. There are no platform fees, no per-request fees, and no bandwidth charges.

### How can I get a tax receipt or VAT invoice?

For international credit card payments processed via Stripe, you can access official Stripe payment receipts and hosted invoices directly from the **Billing History** table at [app.dos.ai/billing](https://app.dos.ai/billing). For corporate billing or Vietnamese tax invoices (e-invoicing via MISA meInvoice), configure your company legal name, tax code, and registered billing address under [Billing Preferences](https://app.dos.ai/billing?tab=tax).
