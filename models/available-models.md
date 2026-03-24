# Available Models

DOS AI serves high-quality open-source LLMs via an OpenAI-compatible API. All models run on dedicated RTX Pro 6000 GPUs with 96 GB VRAM, ensuring fast inference and low latency from our Asia-Southeast 1 region.

## Model Catalog

| Model | Provider | Type | Context Window | Input Price | Output Price |
| ----- | -------- | ---- | -------------- | ----------- | ------------ |
| **Qwen3.5-35B-A3B** | Alibaba | Chat | 128K tokens | $0.15 / 1M tokens | $0.15 / 1M tokens |
| **Llama 3.3 70B** | Meta | Chat | 128K tokens | $0.20 / 1M tokens | $0.20 / 1M tokens |
| **DeepSeek V3** | DeepSeek | Chat | 128K tokens | $0.25 / 1M tokens | $0.25 / 1M tokens |
| **Llama 3.1 8B** | Meta | Chat | 128K tokens | $0.05 / 1M tokens | $0.05 / 1M tokens |

> All prices are in USD. See [Pricing](pricing.md) for details on billing, free tier, and volume discounts.

## Model Details

### Qwen3.5-35B-A3B

Alibaba's Mixture-of-Experts model with 35 billion total parameters and 3 billion active parameters per forward pass. This architecture delivers excellent quality at remarkably low cost and latency, making it our **recommended default model** for most use cases.

- **Best for**: General-purpose chat, code generation, reasoning, multilingual tasks
- **Strengths**: Outstanding cost-efficiency, fast response times, strong multilingual support (especially CJK languages)
- **Model ID**: `dos-ai`

### Llama 3.3 70B

Meta's flagship 70-billion-parameter dense model. Offers top-tier reasoning and instruction-following capabilities.

- **Best for**: Complex reasoning, long-form content, detailed analysis
- **Strengths**: Strong English performance, excellent instruction following, robust safety tuning
- **Model ID**: `llama-3.3-70b`

### DeepSeek V3

DeepSeek's latest Mixture-of-Experts model, known for strong performance across coding, math, and reasoning benchmarks.

- **Best for**: Code generation, mathematical reasoning, structured output
- **Strengths**: Competitive benchmark scores, good at structured/JSON output, strong code capabilities
- **Model ID**: `deepseek-v3`

### Llama 3.1 8B

Meta's efficient 8-billion-parameter model. An excellent choice when you need fast, affordable responses and the task does not require the full capability of a larger model.

- **Best for**: Simple tasks, high-throughput workloads, prototyping, cost-sensitive applications
- **Strengths**: Very low latency, lowest cost, suitable for classification and extraction tasks
- **Model ID**: `llama-3.1-8b`

## Choosing the Right Model

| Use Case | Recommended Model | Why |
| -------- | ----------------- | --- |
| General assistant / chatbot | Qwen3.5-35B-A3B | Best balance of quality, speed, and cost |
| Complex analysis / long documents | Llama 3.3 70B | Strongest reasoning for demanding tasks |
| Code generation / math | DeepSeek V3 | Top coding and math benchmark scores |
| High-volume / low-cost tasks | Llama 3.1 8B | Fastest and cheapest option |
| Multilingual (especially Asian languages) | Qwen3.5-35B-A3B | Superior CJK language performance |

## Listing Models via API

You can retrieve the current list of available models programmatically:

```bash
curl https://api.dos.ai/v1/models \
  -H "Authorization: Bearer YOUR_API_KEY"
```

See the [Models API reference](../api-reference/models.md) for the full response schema.

## Coming Soon

We are continuously evaluating and adding new models. Upcoming additions may include vision models, embedding models, and larger reasoning models. Check back regularly or follow our announcements for updates.
