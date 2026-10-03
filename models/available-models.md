# Available Models

DOS AI serves high-quality open-source LLMs and frontier cloud models via an OpenAI-compatible API. Self-hosted models run on dedicated GPU clusters in Singapore (`asia-southeast1-b`) with NVMe storage and sub-250ms TTFT. Cloud models are served via enterprise partner backends for maximum coverage and high availability.

## Smart Routing (`auto`)

Use `auto` (or `dos-auto`) as the model ID to let the DOS AI Smart Router automatically select the best model for each request. The router classifies incoming prompts into 3 complexity tiers (Simple, Medium, Complex) using task classification, reasoning depth, and context length.

```python
response = client.chat.completions.create(
    model="auto",  # Smart routing dynamically selects the optimal model
    messages=[{"role": "user", "content": "Analyze this multi-step agent workflow..."}],
)
```

### Escalation Hierarchy for Complex Requests

When a request requires advanced multi-step reasoning, complex coding, or autonomous agent loops, the Smart Router escalates to the lowest-priority active tier with verified tool calling and reasoning support:

| Priority | Model | Provider / Backend | Context | Input (per 1M) | Output (per 1M) | Strengths |
| :------- | :---- | :----------------- | :------ | :------------- | :-------------- | :-------- |
| **1 (Primary)** | **GPT-6 Luna** | OpenAI / Azure Foundry | 1.05M | **$0.10** | **$0.50** | Frontier efficiency, fast tool calling, agent loops |
| **2** | **Gemini 3.8 Flash** | Google Cloud | 1M | $0.79 | $3.94 | Sub-second TTFT, multimodal, massive context |
| **3** | **Qwen 3.8 Max** | Alibaba Cloud | 1M | $1.87 | $5.62 | Complex bilingual logic, deep mathematical reasoning |

*Note: Simple and Medium tier requests remain on the in-house self-hosted model (`dos`) to optimize speed and cost.*

---

## Model Catalog

### Self-Hosted (Lowest Latency & In-House Cluster)

| Model | Architecture | Context | Input | Output | Model ID | Primary Region |
| :---- | :----------- | :------ | :---- | :----- | :------- | :------------- |
| **Qwen 3.8 27B Dense** | 27B Dense | 32K | $0.15 / 1M | $0.15 / 1M | `dos`, `dos-ai` | Singapore (`asia-southeast1-b`) |

#### High-Availability Emergency Failover
Self-hosted models feature automated circuit breaker protection with multi-cloud emergency failover:
- **Primary Execution**: Self-hosted vLLM engine on VM `dos` (RTX Pro 6000 Ada / H100 with NVMe).
- **Emergency Priority 1 (Cloudflare Workers AI)**: If vLLM experiences a timeout or 5xx outage, the API Gateway immediately reroutes to Cloudflare Workers AI (`@cf/qwen/qwen3.8-27b`) to absorb the outage with zero downtime.
- **Emergency Priority 2 (Alibaba Cloud Model Studio)**: If Cloudflare is also unreachable, secondary failover routes to Alibaba Cloud.

---

### Frontier & Cloud Models

| Model | Provider | Context | Input Price (1M) | Output Price (1M) | Model ID | Key Capabilities |
| :---- | :------- | :------ | :--------------- | :---------------- | :------- | :--------------- |
| **GPT-6 Luna** | OpenAI / Azure | 1.05M | $0.10 | $0.50 | `gpt-6-luna` | Frontier reasoning, tools, 1.05M context |
| **GPT-6 Sol** | OpenAI / Azure | 1.05M | $2.00 | $10.00 | `gpt-6-sol` | Frontier flagship, deep architectural reasoning |
| **GPT-6 Astra** | OpenAI | 1.05M | $5.00 | $25.00 | `gpt-6-astra` | Next-gen multistep engineering, async tools |
| **Gemini 3.8 Flash** | Google Cloud | 1M | $0.79 | $3.94 | `gemini-3.8-flash` | Hybrid reasoning, vision, sub-second TTFT |
| **DeepSeek V4.1 Flash** | Alibaba / DeepSeek | 1M | $0.14 | $0.28 | `deepseek-v4.1-flash` | MoE, controllable thinking, 1M context |
| **DeepSeek V4 Pro** | DeepSeek / Alibaba | 1M | $2.40 | $4.80 | `deepseek-v4-pro` | Mathematical reasoning, complex agent workflows |
| **Qwen 3.8 27B** | Alibaba Cloud | 1M | $0.50 | $3.00 | `qwen3.8-27b` | Dense reasoning, tool calling, explicit caching |
| **Qwen 3.8 Max** | Alibaba Cloud | 1M | $1.87 | $5.62 | `qwen3.8-max` | Multilingual reasoning, complex logic |
| **Claude Sonnet 5** | Anthropic | 1M | $3.00 | $15.00 | `claude-sonnet-5` | Enterprise agentic coding, 128K output |
| **Claude Opus 5** | Anthropic | 1M | $15.00 | $75.00 | `claude-opus-5` | Flagship reasoning, long-horizon tasks |

> All prices are in USD and strictly adhere to DOS AI's **Zero-Markup Policy** (100% of provider published list price). Check `GET /v1/catalog` or the [web dashboard](https://app.dos.ai/models) for real-time status.

---

### Embedding Models

| Model | Provider | Dimensions | Max Input Tokens | Model ID |
| :---- | :------- | :--------- | :--------------- | :------- |
| **Qwen3-Embedding-4B** | Alibaba / Self-hosted | 2560 | 32,768 | `qwen3-embedding-4b` |
| **BGE-Large-EN-v1.5** | Cloudflare / BAAI | 1024 | 512 | `baai/bge-large-en-v1.5` |

See the [Embeddings API reference](../api-reference/embeddings.md) for request/response formats and code examples.

---

## Model Details

### GPT-6 Luna
OpenAI's latest cost-efficient frontier reasoning model. Features state-of-the-art instruction following, coding, structured JSON outputs, and native tool-calling agent capabilities at more than 50% lower cost than previous generations ($0.10 in / $0.50 out).

- **Best for**: Autonomous agent loops, function calling, high-volume production chat, code assistance
- **Strengths**: 1.05M context window, exceptional price-to-performance, Azure Foundry low-latency hosting
- **Model ID**: `gpt-6-luna`

### Qwen 3.8 27B Dense (`dos`)
The foundational in-house model driving DOSClaw agents. Hosted on dedicated GPUs in Singapore, offering sub-250ms TTFT with consistent low-latency response times.

- **Best for**: Agent turn executions, customer support, fast interactive chat, CJK bilingual tasks
- **Strengths**: Sub-250ms latency, high instruction following, automated Cloudflare emergency failover
- **Model ID**: `dos`, `dos-ai`

### Gemini 3.8 Flash
Google's high-speed multimodal reasoning model with hybrid thinking, 1M context window, and sub-second execution.

- **Best for**: Real-time document parsing, large multimodal payloads, high-throughput applications
- **Strengths**: 1M context, sub-second TTFT, native audio/video/image comprehension
- **Model ID**: `gemini-3.8-flash`

---

## Choosing the Right Model

| Use Case | Recommended Model | Why |
| :------- | :---------------- | :-- |
| **Let DOS AI decide** | `auto` | Smart Router dynamically matches task complexity and cost |
| **Fastest agent execution & lowest latency** | `dos` | Self-hosted 27B dense cluster in Singapore, sub-250ms TTFT |
| **Ultra-low cost frontier reasoning** | `gpt-6-luna` | $0.10 / $0.50 per million tokens with 1.05M context |
| **Sub-second large document analysis** | `gemini-3.8-flash` | 1M context window, Google Cloud high-throughput backend |
| **Enterprise agentic coding & architecture** | `claude-sonnet-5` | Frontier coding quality with 128K max output tokens |
| **Deep mathematical reasoning** | `deepseek-v4-pro` | SOTA chain-of-thought problem solving |

---

## Listing Models via API

Retrieve the current list of available models programmatically:

```bash
curl https://api.dos.ai/v1/models \
  -H "Authorization: Bearer $DOS_API_KEY"
```

For the full retail catalog including pricing, context lengths, and capability flags:

```bash
curl https://api.dos.ai/v1/catalog \
  -H "Authorization: Bearer $DOS_API_KEY"
```
