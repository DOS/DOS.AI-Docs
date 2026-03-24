# DOS AI

**Fast, affordable AI inference for open-source models.**

DOS AI is an inference platform that lets you run leading open-source language models through a simple, OpenAI-compatible API. No GPU management, no infrastructure headaches -- just an API key and a few lines of code.

## Why DOS AI?

- **OpenAI-compatible** -- Swap your base URL and you're done. Works with the OpenAI Python SDK, Node.js SDK, LangChain, LlamaIndex, and any HTTP client.
- **Low latency** -- Models served on dedicated GPUs with optimized inference (vLLM). No cold starts, no queues.
- **Pay-as-you-go** -- Only pay for the tokens you use. Every new account gets **$5 in free credits** to get started.
- **Open-source models** -- Access the best open-source models without managing your own infrastructure.

## Quick start

Get up and running in under a minute:

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.dos.ai/v1",
    api_key="dos_sk_...",  # Get your key at app.dos.ai
)

response = client.chat.completions.create(
    model="dos-ai",
    messages=[
        {"role": "user", "content": "Explain quantum computing in one paragraph."}
    ],
)

print(response.choices[0].message.content)
```

## Documentation

| Section | Description |
| --- | --- |
| [Quickstart](getting-started/quickstart.md) | Create an account, get an API key, and make your first request |
| [Authentication](getting-started/authentication.md) | API key management, rate limits, and security best practices |
| [OpenAI Compatibility](getting-started/openai-compatibility.md) | Migration guide and compatibility details |

## Available models

| Model ID | Base model | Context length | Pricing |
| --- | --- | --- | --- |
| `dos-ai` | Qwen3.5-35B-A3B | 32,768 tokens | See [dashboard](https://app.dos.ai) |

More models are added regularly. Check the [models endpoint](https://api.dos.ai/v1/models) or your dashboard for the latest list.

## Links

- **Dashboard**: [app.dos.ai](https://app.dos.ai)
- **API base URL**: `https://api.dos.ai/v1`
- **Status**: [status.dos.ai](https://status.dos.ai)
