# Caching & Context Optimization

DOS AI provides a multi-layer caching and context optimization system designed to minimize latency, reduce token consumption, and cut API costs. This includes:

1. **L1 Response Caching**: Instant, sub-millisecond serving of identical requests at **0 tokens and $0 cost**.
2. **Upstream Prompt Caching Pass-Through**: Full preservation of upstream prompt caching markers with **100% discount pass-through** (zero-markup).
3. **Context & Prompt Compression**: Opt-in payload compaction for large contexts.

---

## 1. L1 Response Caching

DOS AI caches completed inference responses in a fast, in-memory LRU layer. When an identical request arrives, it is served immediately without calling the upstream model.

### Key Benefits

* **Zero Token Cost**: Cache hits never consume tokens or billing credits.
* **Sub-Millisecond Latency**: Responses return in < 1ms directly from the gateway edge.
* **Streaming & Non-Streaming**: Both standard JSON completions and streaming SSE (`stream: true`) responses are supported via stream replay.
* **Strict Tenant Isolation**: Powered by `TenantCacheScope`, cached entries are strictly isolated to your authenticated identity (User ID, API Key ID, Team ID, Product, and BYOK credential). Cache items are never shared across tenants.

### Cache Controls (`X-DOS-Cache`)

| `X-DOS-Cache` Request Header | Behavior |
| ---------------------------- | -------- |
| *(omitted)* | **Default-on for deterministic requests** (`temperature: 0`). Non-deterministic requests bypass the cache. |
| `on` | **Force caching on**, even when `temperature > 0`. Useful when you want fixed responses for identical prompts. |
| `off` | **Bypass caching entirely** — guarantees a fresh generation directly from the upstream provider. |

### Response Headers

| Header | Value / Meaning |
| ------ | --------------- |
| `X-DOS-Cache` | `hit` (served free from cache) or `miss` (fresh generation from model). |
| `X-Provider` | Displays `cache` on a cache hit instead of the upstream provider name. |

### What Counts as "The Same Request"

The cache key is computed as the SHA-256 hash of:
1. The authenticated tenant scope (`TenantCacheScope`),
2. The endpoint path (e.g. `/v1/chat/completions`),
3. The canonical JSON request body.

Field ordering does not matter, and non-output metadata fields (`stream`, `stream_options`, `user`, `metadata`) are stripped before hashing. All generation parameters (`model`, `messages`, `temperature`, `max_tokens`, `tools`) form part of the key.

### Example: Using Response Cache

```bash
# First request: Cache miss (fresh model call)
curl -i https://api.dos.ai/v1/chat/completions \
  -H "Authorization: Bearer dos_sk_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "deepseek/deepseek-chat", "temperature": 0, "messages": [{"role": "user", "content": "What is the capital of Vietnam?"}]}'

# Second request: Cache hit (instant <1ms, 0 tokens, $0 cost)
# Response header: X-DOS-Cache: hit, X-Provider: cache
curl -i https://api.dos.ai/v1/chat/completions \
  -H "Authorization: Bearer dos_sk_your_key" \
  -H "Content-Type: application/json" \
  -d '{"model": "deepseek/deepseek-chat", "temperature": 0, "messages": [{"role": "user", "content": "What is the capital of Vietnam?"}]}'
```

---

## 2. Upstream Prompt Caching & Zero-Markup Guarantee

When working with long prompts, multi-turn conversations, or complex agent workflows, major foundation models (Anthropic, OpenAI, DeepSeek, Google Gemini) offer **Prompt Caching** (Context Caching).

DOS AI fully preserves these caching capabilities and enforces a strict **Zero-Markup Policy**: **100% of provider cache discounts are passed directly to you**.

### Supported Provider Caching Protocols

* **Anthropic Claude**: Full support for explicit breakpoint markers:
  ```json
  "content": [
    {
      "type": "text",
      "text": "Your large system prompt, documentation, or codebase...",
      "cache_control": {"type": "ephemeral"}
    }
  ]
  ```
  * Up to 4 cache breakpoints per request.
  * **90% discount** on cached input tokens passed through with zero markup.
* **OpenAI & Azure OpenAI**: Automatic prefix caching on prompts longer than 1,024 tokens.
  * **50% discount** on cached input tokens passed through.
* **DeepSeek & Alibaba DashScope**: Automatic context caching for repeated prefix blocks.
* **Google Gemini**: Implicit context caching for long context windows.

### Verifying Cache Usage

The response `usage` block explicitly reports cached token savings:

```json
{
  "usage": {
    "prompt_tokens": 12500,
    "completion_tokens": 340,
    "total_tokens": 12840,
    "prompt_tokens_details": {
      "cached_tokens": 12000
    },
    "cost": 0.00350
  }
}
```

In the example above, you are billed for 12,000 tokens at the discounted cached input rate, and only 500 tokens at the standard base input rate.

---

## 3. Prompt Compression (`X-DOS-Prompt-Compression`)

For applications sending very large conversation histories or repetitive data, DOS AI includes a multi-layer context compression engine powered by DOSRouter.

### How it Works

When opted in:
1. It analyzes requests longer than 5,000 characters.
2. It deduplicates repeated messages, strips redundant whitespace in code and text, and compacts verbose JSON blocks.
3. It retains 100% semantic fidelity and formatting tags.

### Enabling Prompt Compression

Because byte-exact prefix caching depends on identical input bytes, prompt compression is **opt-in** to avoid invalidating upstream cache hashes:

```bash
curl -i https://api.dos.ai/v1/chat/completions \
  -H "Authorization: Bearer dos_sk_your_key" \
  -H "X-DOS-Prompt-Compression: true" \
  -H "Content-Type: application/json" \
  -d '{"model": "qwen/qwen-2.5-coder-32b-instruct", "messages": [...large messages...]}'
```

When compression is applied, the response includes an efficiency header:

```http
X-DOS-Compression-Savings: 3412
```
*(Indicates that 3,412 redundant characters were pruned before calling the model).*
