# Streaming

Streaming lets you receive the model's response token-by-token as it is generated, rather than waiting for the entire response to complete. This dramatically reduces perceived latency -- the user sees output within milliseconds instead of waiting seconds for a full response.

## Why Use Streaming

- **Faster time-to-first-token.** The user sees output almost immediately.
- **Better UX for long responses.** Progressive rendering feels more responsive than a loading spinner.
- **Real-time applications.** Chat interfaces, live coding assistants, and interactive tools all benefit from streaming.
- **Memory efficiency.** Process tokens incrementally without buffering the entire response.

## How It Works

Set `stream: true` in your request. The API responds with a stream of **Server-Sent Events (SSE)** instead of a single JSON response.

Each event is a line prefixed with `data: ` containing a JSON chunk. The stream ends with `data: [DONE]`.

```
data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"The"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" capital"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" of"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" France"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" is"},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" Paris."},"finish_reason":null}]}

data: {"id":"chatcmpl-abc","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}

data: [DONE]
```

### Key Differences from Non-streaming

| Aspect | Non-streaming | Streaming |
|--------|--------------|-----------|
| Response type | Single JSON object | Stream of SSE events |
| Object type | `chat.completion` | `chat.completion.chunk` |
| Message field | `message` | `delta` (incremental) |
| Content | Complete string | Token-by-token fragments |
| Usage stats | Included in response | Included in the final chunk (with `stream_options`) |

### The `delta` Object

In streaming, each chunk contains a `delta` instead of a `message`. The delta holds only the new content since the last chunk:

- First chunk: `delta` has `role: "assistant"` (and optionally the first content token).
- Middle chunks: `delta` has `content` with the next token(s).
- Final chunk: `delta` is empty `{}`, and `finish_reason` is set (e.g., `"stop"` or `"tool_calls"`).

## Getting Usage Statistics

By default, streaming responses do not include token usage. To receive usage data, set `stream_options`:

```json
{
  "model": "dos-ai",
  "messages": [{"role": "user", "content": "Hello!"}],
  "stream": true,
  "stream_options": {
    "include_usage": true
  }
}
```

The final chunk before `[DONE]` will include a `usage` field:

```json
{
  "id": "chatcmpl-abc",
  "choices": [],
  "usage": {
    "prompt_tokens": 9,
    "completion_tokens": 12,
    "total_tokens": 21
  }
}
```

## Code Examples

### cURL

```bash
curl https://api.dos.ai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dos_sk_your_api_key_here" \
  -N \
  -d '{
    "model": "dos-ai",
    "messages": [
      {"role": "user", "content": "Write a short poem about the ocean."}
    ],
    "stream": true
  }'
```

The `-N` flag disables output buffering so you see tokens as they arrive.

### Python (OpenAI SDK -- Synchronous)

```python
from openai import OpenAI

client = OpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
)

stream = client.chat.completions.create(
    model="dos-ai",
    messages=[
        {"role": "user", "content": "Write a short poem about the ocean."}
    ],
    stream=True,
)

for chunk in stream:
    content = chunk.choices[0].delta.content
    if content:
        print(content, end="", flush=True)

print()  # Newline after stream completes
```

### Python (OpenAI SDK -- Async)

```python
import asyncio
from openai import AsyncOpenAI

client = AsyncOpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
)


async def main():
    stream = await client.chat.completions.create(
        model="dos-ai",
        messages=[
            {"role": "user", "content": "Write a short poem about the ocean."}
        ],
        stream=True,
    )

    async for chunk in stream:
        content = chunk.choices[0].delta.content
        if content:
            print(content, end="", flush=True)

    print()


asyncio.run(main())
```

### Python (requests -- Manual SSE Parsing)

For cases where you cannot use the OpenAI SDK:

```python
import json
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
            {"role": "user", "content": "Write a short poem about the ocean."}
        ],
        "stream": True,
    },
    stream=True,
)

for line in response.iter_lines():
    if not line:
        continue

    line = line.decode("utf-8")

    if line.startswith("data: "):
        data = line[6:]  # Remove "data: " prefix

        if data == "[DONE]":
            break

        chunk = json.loads(data)
        content = chunk["choices"][0]["delta"].get("content", "")
        if content:
            print(content, end="", flush=True)

print()
```

### JavaScript (OpenAI SDK)

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "dos_sk_your_api_key_here",
  baseURL: "https://api.dos.ai/v1",
});

const stream = await client.chat.completions.create({
  model: "dos-ai",
  messages: [
    { role: "user", content: "Write a short poem about the ocean." },
  ],
  stream: true,
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content;
  if (content) {
    process.stdout.write(content);
  }
}

console.log();
```

### JavaScript (fetch -- Browser/Edge)

For browser-based applications or edge runtimes where the OpenAI SDK is not available:

```javascript
async function streamChat(messages) {
  const response = await fetch("https://api.dos.ai/v1/chat/completions", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      Authorization: "Bearer dos_sk_your_api_key_here",
    },
    body: JSON.stringify({
      model: "dos-ai",
      messages,
      stream: true,
    }),
  });

  if (!response.ok) {
    throw new Error(`HTTP ${response.status}: ${response.statusText}`);
  }

  const reader = response.body.getReader();
  const decoder = new TextDecoder();
  let buffer = "";

  while (true) {
    const { done, value } = await reader.read();
    if (done) break;

    buffer += decoder.decode(value, { stream: true });

    // Process complete lines
    const lines = buffer.split("\n");
    buffer = lines.pop(); // Keep incomplete line in buffer

    for (const line of lines) {
      const trimmed = line.trim();
      if (!trimmed || !trimmed.startsWith("data: ")) continue;

      const data = trimmed.slice(6);
      if (data === "[DONE]") return;

      const chunk = JSON.parse(data);
      const content = chunk.choices[0]?.delta?.content;
      if (content) {
        // Append to your UI element
        document.getElementById("output").textContent += content;
      }
    }
  }
}

// Usage
await streamChat([
  { role: "user", content: "Write a short poem about the ocean." },
]);
```

### React (Next.js with Vercel AI SDK)

For Next.js applications, the Vercel AI SDK provides a streamlined experience:

```typescript
// app/api/chat/route.ts
import { createOpenAI } from "@ai-sdk/openai";
import { streamText } from "ai";

const dosai = createOpenAI({
  apiKey: process.env.DOS_AI_API_KEY,
  baseURL: "https://api.dos.ai/v1",
});

export async function POST(req: Request) {
  const { messages } = await req.json();

  const result = streamText({
    model: dosai("dos-ai"),
    messages,
  });

  return result.toUIMessageStreamResponse();
}
```

```tsx
// app/page.tsx
"use client";

import { useChat } from "@ai-sdk/react";

export default function Chat() {
  const { messages, input, setInput, sendMessage } = useChat({
    api: "/api/chat",
  });

  return (
    <div>
      {messages.map((m) => (
        <div key={m.id}>
          <strong>{m.role}:</strong> {m.content}
        </div>
      ))}
      <form
        onSubmit={(e) => {
          e.preventDefault();
          sendMessage({ text: input });
          setInput("");
        }}
      >
        <input value={input} onChange={(e) => setInput(e.target.value)} />
        <button type="submit">Send</button>
      </form>
    </div>
  );
}
```

## Streaming with Function Calls

When the model makes a tool call during streaming, the chunks contain `delta.tool_calls` instead of `delta.content`. The function name and arguments arrive incrementally:

```python
from openai import OpenAI

client = OpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
)

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get the current weather for a city.",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"},
                },
                "required": ["city"],
            },
        },
    },
]

stream = client.chat.completions.create(
    model="dos-ai",
    messages=[{"role": "user", "content": "What is the weather in Hanoi?"}],
    tools=tools,
    stream=True,
)

# Accumulate tool call data from chunks
tool_calls = {}
current_content = ""

for chunk in stream:
    delta = chunk.choices[0].delta

    if delta.content:
        current_content += delta.content
        print(delta.content, end="", flush=True)

    if delta.tool_calls:
        for tc in delta.tool_calls:
            idx = tc.index
            if idx not in tool_calls:
                tool_calls[idx] = {
                    "id": tc.id,
                    "function": {"name": "", "arguments": ""},
                }
            if tc.function.name:
                tool_calls[idx]["function"]["name"] += tc.function.name
            if tc.function.arguments:
                tool_calls[idx]["function"]["arguments"] += tc.function.arguments

# After the stream, tool_calls contains the complete function call data
for idx, tc in tool_calls.items():
    print(f"\nTool call: {tc['function']['name']}({tc['function']['arguments']})")
```

## Error Handling

### Connection Errors

Streaming connections can be interrupted by network issues. Always handle connection errors and implement reconnection logic:

```python
from openai import OpenAI, APIConnectionError, APITimeoutError

client = OpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
)


def stream_with_retry(messages, max_retries=3):
    for attempt in range(max_retries):
        try:
            stream = client.chat.completions.create(
                model="dos-ai",
                messages=messages,
                stream=True,
            )

            full_response = ""
            for chunk in stream:
                content = chunk.choices[0].delta.content
                if content:
                    full_response += content
                    print(content, end="", flush=True)

            print()
            return full_response

        except (APIConnectionError, APITimeoutError) as e:
            print(f"\nConnection error (attempt {attempt + 1}): {e}")
            if attempt == max_retries - 1:
                raise

        except Exception as e:
            print(f"\nUnexpected error: {e}")
            raise
```

### Incomplete Streams

If the stream ends unexpectedly (no `[DONE]` event), check the last chunk's `finish_reason`:

- `"stop"` -- normal completion.
- `"length"` -- hit the `max_tokens` limit. Increase `max_tokens` or continue the conversation.
- `"tool_calls"` -- the model wants to call a function. Handle the tool call and continue.
- `null` -- stream was interrupted. Retry the request.

### Timeouts

For long-running streams, configure appropriate timeouts:

```python
client = OpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
    timeout=120.0,  # 2 minutes
)
```

```javascript
const client = new OpenAI({
  apiKey: "dos_sk_your_api_key_here",
  baseURL: "https://api.dos.ai/v1",
  timeout: 120000, // 2 minutes in milliseconds
});
```

## Collecting the Full Response

If you need the complete response text (e.g., for logging or saving to a database), accumulate it during streaming:

```python
stream = client.chat.completions.create(
    model="dos-ai",
    messages=[{"role": "user", "content": "Tell me a story."}],
    stream=True,
    stream_options={"include_usage": True},
)

full_text = ""
usage = None

for chunk in stream:
    if chunk.choices:
        content = chunk.choices[0].delta.content
        if content:
            full_text += content
            print(content, end="", flush=True)

    if chunk.usage:
        usage = chunk.usage

print()
print(f"\nTotal tokens: {usage.total_tokens if usage else 'N/A'}")

# Save full_text to your database
```

## Next Steps

- [Chat Completions](chat-completions.md) -- learn the basics of the API.
- [Function Calling](function-calling.md) -- let the model call external tools.
- [Structured Outputs](structured-outputs.md) -- get reliable JSON from the model.
