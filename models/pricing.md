# Pricing

DOS AI uses a simple **pay-as-you-go** pricing model. You only pay for the tokens you use, with no minimum commitments, no monthly fees, and no hidden charges.

## Zero-Markup Policy

DOS AI operates on a strict **Zero-Markup Policy**:
- **100% Provider Published List Price**: We sell cloud and third-party models at exactly 100% of the upstream provider's official published list price.
- **No Hidden Fees or Surcharges**: You never pay more for tokens than you would by calling providers directly.
- **Single Transparent Bill**: Consolidate multiple model providers into one unified account, API key, and balance without paying markup penalties.

---

## Free Tier

Every new account receives **$5.00 in free credits** to get started. This is enough for substantial experimentation and prototyping before you need to add funds.

| Model | Approximate Free Usage |
| ----- | ---------------------- |
| Qwen3.5-35B-A3B | ~33 million tokens |
| Llama 3.3 70B | ~25 million tokens |
| DeepSeek V3 | ~20 million tokens |
| Llama 3.1 8B | ~100 million tokens |

> Free credits do not expire. No credit card is required to start.

---

## Per-Token Pricing

Pricing is calculated per **1 million tokens** (input, output, and cached input). All rates are database-driven and strictly follow official provider published prices.

| Model | Provider | Input Price (per 1M) | Output Price (per 1M) | Cached Input (per 1M) |
| :---- | :------- | :------------------- | :-------------------- | :-------------------- |
| **Qwen3.5-35B-A3B** (default) | Alibaba / Self-hosted | $0.15 | $0.15 | — |
| **DeepSeek V4 Pro** | DeepSeek / Alibaba | $2.40 | $4.80 | $0.20 |
| **Qwen3.8 27B** | Alibaba | $0.50 | $3.00 | $0.10 |
| **Llama 4 Maverick 17B-128E** | Meta / DeepInfra | $0.17 | $0.66 | — |
| **Llama 4 Scout 17B-16E** | Meta / DeepInfra | $0.11 | $0.38 | — |
| **DeepSeek V3** | DeepSeek | $0.25 | $0.25 | — |
| **Llama 3.3 70B** | Meta | $0.20 | $0.20 | — |
| **Llama 3.1 8B** | Meta | $0.05 | $0.05 | — |

> Prices are statically database-driven from Supabase catalog and updated strictly via migrations. Check the [dashboard](https://app.dos.ai/models) or `GET /v1/models` for real-time account pricing.

### Prompt Caching Savings

For supported models (including DeepSeek V4 Pro and Qwen 3.8), repeated prompt prefixes automatically benefit from prompt caching:
- **DeepSeek V4 Pro**: Cached input tokens are billed at **$0.20 / 1M tokens** (over 90% savings compared to standard input).
- **Qwen 3.8 27B**: Cached input tokens are billed at **$0.10 / 1M tokens** (80% savings).

---

### What is a Token?

A token is roughly 3-4 characters of English text, or about 0.75 words. For example:

- "Hello, world!" = approximately 4 tokens
- A typical 500-word blog post = approximately 650-700 tokens
- A full 128K context window = approximately 96,000 words

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
    }
  }
}
```

You can also view historical usage and spending breakdowns on the [dashboard](https://app.dos.ai).

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
