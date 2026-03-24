# Chat Completions

The Chat Completions API is the primary way to interact with DOS AI models. It follows the OpenAI-compatible format, so you can use existing OpenAI SDKs and tools with minimal changes.

## Base URL

```
https://api.dos.ai/v1
```

## Authentication

All requests require an API key passed in the `Authorization` header:

```
Authorization: Bearer dos_sk_your_api_key_here
```

## Basic Request

A chat completion request consists of a list of messages and a model identifier. The model generates a response based on the conversation history.

### Request Format

```json
POST /v1/chat/completions
Content-Type: application/json
Authorization: Bearer dos_sk_your_api_key_here

{
  "model": "dos-ai",
  "messages": [
    { "role": "user", "content": "What is the capital of France?" }
  ]
}
```

### Response Format

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1711500000,
  "model": "dos-ai",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "The capital of France is Paris."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 14,
    "completion_tokens": 8,
    "total_tokens": 22
  }
}
```

## Message Roles

Each message in the `messages` array has a `role` and `content`. The API supports four roles:

| Role | Description |
|------|-------------|
| `system` | Sets the behavior and personality of the assistant. Placed at the beginning of the conversation. |
| `user` | Messages from the end user. |
| `assistant` | Previous responses from the model. Used for multi-turn context. |
| `tool` | Results from tool/function calls. See [Function Calling](function-calling.md). |

### System Message

Use the system message to instruct the model on how to behave:

```json
{
  "model": "dos-ai",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful coding assistant. Always include code examples in your answers. Use Python unless the user specifies a different language."
    },
    {
      "role": "user",
      "content": "How do I read a CSV file?"
    }
  ]
}
```

## Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `model` | string | *required* | Model ID to use (e.g., `dos-ai`). |
| `messages` | array | *required* | List of messages in the conversation. |
| `temperature` | float | 0.7 | Sampling temperature between 0 and 2. Lower values make output more deterministic. |
| `max_tokens` | integer | model default | Maximum number of tokens to generate in the response. |
| `top_p` | float | 1.0 | Nucleus sampling threshold. Only tokens with cumulative probability up to `top_p` are considered. |
| `frequency_penalty` | float | 0.0 | Penalizes tokens based on how frequently they appear (range: -2.0 to 2.0). |
| `presence_penalty` | float | 0.0 | Penalizes tokens based on whether they have appeared at all (range: -2.0 to 2.0). |
| `stop` | string or array | null | Up to 4 sequences where the model will stop generating. |
| `stream` | boolean | false | If true, returns a stream of server-sent events. See [Streaming](streaming.md). |
| `n` | integer | 1 | Number of completions to generate for each prompt. |
| `response_format` | object | null | Force a specific output format. See [Structured Outputs](structured-outputs.md). |
| `tools` | array | null | List of tools the model may call. See [Function Calling](function-calling.md). |

### Temperature vs Top-p

- **Temperature** controls randomness. `0` is nearly deterministic, `2` is highly random.
- **Top-p** controls diversity by limiting the token pool. `0.1` means only the top 10% probability mass is considered.

It is generally recommended to adjust one or the other, not both simultaneously.

```json
{
  "model": "dos-ai",
  "messages": [{ "role": "user", "content": "Write a haiku about AI." }],
  "temperature": 1.2,
  "max_tokens": 50
}
```

## Multi-turn Conversations

To maintain context across multiple exchanges, include previous messages in the request. The model does not retain state between requests -- you must send the full conversation history each time.

```json
{
  "model": "dos-ai",
  "messages": [
    { "role": "system", "content": "You are a math tutor." },
    { "role": "user", "content": "What is the derivative of x^2?" },
    { "role": "assistant", "content": "The derivative of x^2 is 2x." },
    { "role": "user", "content": "What about x^3?" }
  ]
}
```

The model sees the full conversation and can respond contextually: "The derivative of x^3 is 3x^2."

## Code Examples

### cURL

```bash
curl https://api.dos.ai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dos_sk_your_api_key_here" \
  -d '{
    "model": "dos-ai",
    "messages": [
      {"role": "system", "content": "You are a helpful assistant."},
      {"role": "user", "content": "Explain quantum computing in simple terms."}
    ],
    "temperature": 0.7,
    "max_tokens": 500
  }'
```

### Python (OpenAI SDK)

The easiest way to use DOS AI in Python is with the official OpenAI SDK, pointed at the DOS AI base URL:

```python
from openai import OpenAI

client = OpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
)

response = client.chat.completions.create(
    model="dos-ai",
    messages=[
        {"role": "system", "content": "You are a helpful assistant."},
        {"role": "user", "content": "Explain quantum computing in simple terms."},
    ],
    temperature=0.7,
    max_tokens=500,
)

print(response.choices[0].message.content)
```

### Python (requests)

If you prefer not to use the OpenAI SDK:

```python
import requests

response = requests.post(
    "https://api.dos.ai/v1/chat/completions",
    headers={
        "Content-Type": "application/json",
        "Authorization": "Bearer dos_sk_your_api_key_here",
    },
    json={
        "model": "dos-ai",
        "messages": [
            {"role": "system", "content": "You are a helpful assistant."},
            {"role": "user", "content": "Explain quantum computing in simple terms."},
        ],
        "temperature": 0.7,
        "max_tokens": 500,
    },
)

data = response.json()
print(data["choices"][0]["message"]["content"])
```

### JavaScript (Node.js)

Using the OpenAI Node.js SDK:

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "dos_sk_your_api_key_here",
  baseURL: "https://api.dos.ai/v1",
});

const response = await client.chat.completions.create({
  model: "dos-ai",
  messages: [
    { role: "system", content: "You are a helpful assistant." },
    { role: "user", content: "Explain quantum computing in simple terms." },
  ],
  temperature: 0.7,
  max_tokens: 500,
});

console.log(response.choices[0].message.content);
```

### JavaScript (fetch)

Using the native `fetch` API:

```javascript
const response = await fetch("https://api.dos.ai/v1/chat/completions", {
  method: "POST",
  headers: {
    "Content-Type": "application/json",
    Authorization: "Bearer dos_sk_your_api_key_here",
  },
  body: JSON.stringify({
    model: "dos-ai",
    messages: [
      { role: "system", content: "You are a helpful assistant." },
      { role: "user", content: "Explain quantum computing in simple terms." },
    ],
    temperature: 0.7,
    max_tokens: 500,
  }),
});

const data = await response.json();
console.log(data.choices[0].message.content);
```

## Error Handling

The API returns standard HTTP status codes and a JSON error body:

| Status Code | Meaning |
|-------------|---------|
| `400` | Bad request -- malformed JSON or invalid parameters. |
| `401` | Unauthorized -- missing or invalid API key. |
| `402` | Insufficient credits -- top up your balance. |
| `429` | Rate limit exceeded -- slow down and retry after the indicated period. |
| `500` | Internal server error -- retry with exponential backoff. |
| `503` | Service unavailable -- the model is temporarily overloaded. |

Error response format:

```json
{
  "error": {
    "message": "Invalid API key provided.",
    "type": "authentication_error",
    "code": "invalid_api_key"
  }
}
```

### Retry Strategy

For `429` and `5xx` errors, implement exponential backoff:

```python
import time
import requests

def chat_with_retry(payload, max_retries=3):
    for attempt in range(max_retries):
        response = requests.post(
            "https://api.dos.ai/v1/chat/completions",
            headers={
                "Content-Type": "application/json",
                "Authorization": "Bearer dos_sk_your_api_key_here",
            },
            json=payload,
        )

        if response.status_code == 200:
            return response.json()

        if response.status_code in (429, 500, 503):
            wait = 2 ** attempt
            print(f"Retrying in {wait}s (status {response.status_code})...")
            time.sleep(wait)
            continue

        # Non-retryable error
        response.raise_for_status()

    raise Exception("Max retries exceeded")
```

## Available Models

| Model ID | Description |
|----------|-------------|
| `dos-ai` | Qwen3.5-35B-A3B -- fast, efficient, recommended for most tasks. |

Check the [Models](/api-reference/models.md) endpoint for the current list of available models.

## Next Steps

- [Streaming](streaming.md) -- receive tokens in real time as they are generated.
- [Function Calling](function-calling.md) -- let the model call external tools and APIs.
- [Structured Outputs](structured-outputs.md) -- get reliable JSON responses.
