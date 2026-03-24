# OpenAI Compatibility

DOS AI implements the OpenAI API specification. If your application already uses the OpenAI API, you can switch to DOS AI by changing two lines of configuration -- no code rewrite required.

## Drop-in replacement

The only changes needed are `base_url` (or `baseURL`) and `api_key`:

### Python

```python
from openai import OpenAI

# Before (OpenAI)
# client = OpenAI(api_key="sk-...")

# After (DOS AI)
client = OpenAI(
    base_url="https://api.dos.ai/v1",
    api_key="dos_sk_...",
)

# Everything else stays the same
response = client.chat.completions.create(
    model="dos-ai",
    messages=[{"role": "user", "content": "Hello!"}],
    temperature=0.7,
    max_tokens=256,
)
```

### JavaScript / TypeScript

```javascript
import OpenAI from "openai";

// Before (OpenAI)
// const client = new OpenAI({ apiKey: "sk-..." });

// After (DOS AI)
const client = new OpenAI({
  baseURL: "https://api.dos.ai/v1",
  apiKey: "dos_sk_...",
});

// Everything else stays the same
const response = await client.chat.completions.create({
  model: "dos-ai",
  messages: [{ role: "user", content: "Hello!" }],
  temperature: 0.7,
  max_tokens: 256,
});
```

### cURL / HTTP

Replace `https://api.openai.com` with `https://api.dos.ai` and update your API key:

```bash
curl https://api.dos.ai/v1/chat/completions \
  -H "Authorization: Bearer dos_sk_..." \
  -H "Content-Type: application/json" \
  -d '{
    "model": "dos-ai",
    "messages": [{"role": "user", "content": "Hello!"}],
    "temperature": 0.7,
    "max_tokens": 256
  }'
```

## Supported endpoints

| Endpoint | Method | Description |
| --- | --- | --- |
| `/v1/chat/completions` | POST | Chat completions (single and streaming) |
| `/v1/models` | GET | List available models |

## Supported parameters

The `/v1/chat/completions` endpoint supports the following request parameters:

| Parameter | Type | Description |
| --- | --- | --- |
| `model` | string | **Required.** Model ID (e.g., `dos-ai`) |
| `messages` | array | **Required.** Conversation messages (`role` + `content`) |
| `temperature` | float | Sampling temperature (0.0 -- 2.0). Default: 1.0 |
| `top_p` | float | Nucleus sampling threshold. Default: 1.0 |
| `max_tokens` | integer | Maximum tokens to generate |
| `stream` | boolean | Enable server-sent events streaming. Default: false |
| `stop` | string or array | Stop sequence(s) |
| `frequency_penalty` | float | Penalize repeated tokens (-2.0 -- 2.0). Default: 0.0 |
| `presence_penalty` | float | Penalize tokens already present (-2.0 -- 2.0). Default: 0.0 |
| `n` | integer | Number of completions to generate. Default: 1 |
| `seed` | integer | Seed for deterministic sampling |

## Framework compatibility

Since DOS AI follows the OpenAI specification, it works out of the box with popular frameworks and libraries:

### LangChain

```python
from langchain_openai import ChatOpenAI

llm = ChatOpenAI(
    base_url="https://api.dos.ai/v1",
    api_key="dos_sk_...",
    model="dos-ai",
)

response = llm.invoke("Explain recursion in one sentence.")
print(response.content)
```

### LlamaIndex

```python
from llama_index.llms.openai_like import OpenAILike

llm = OpenAILike(
    api_base="https://api.dos.ai/v1",
    api_key="dos_sk_...",
    model="dos-ai",
    is_chat_model=True,
)
```

### Vercel AI SDK

```typescript
import { createOpenAI } from "@ai-sdk/openai";
import { generateText } from "ai";

const dosai = createOpenAI({
  baseURL: "https://api.dos.ai/v1",
  apiKey: "dos_sk_...",
});

const { text } = await generateText({
  model: dosai("dos-ai"),
  prompt: "Explain recursion in one sentence.",
});
```

## What's different from OpenAI

While we aim for full compatibility, there are a few differences to be aware of:

| Feature | Status | Notes |
| --- | --- | --- |
| Chat completions | Supported | Full compatibility including streaming |
| Model names | Different | Use DOS AI model IDs (e.g., `dos-ai` instead of `gpt-4o`) |
| Function calling / tools | Supported | Works with vLLM tool-calling implementation |
| JSON mode | Supported | Set `response_format: {"type": "json_object"}` |
| Vision (image inputs) | Model-dependent | Supported if the underlying model handles vision |
| Embeddings | Not yet available | Coming soon |
| Image generation | Not available | Use a dedicated image generation service |
| Audio / TTS / STT | Not available | Not on the roadmap |
| Assistants API | Not available | Use chat completions directly |
| Batch API | Not yet available | Coming soon |

## Migration checklist

Switching from OpenAI to DOS AI:

- [ ] Sign up at [app.dos.ai](https://app.dos.ai) and create an API key
- [ ] Update `base_url` / `baseURL` to `https://api.dos.ai/v1`
- [ ] Update `api_key` / `apiKey` to your `dos_sk_...` key
- [ ] Update the `model` parameter to a DOS AI model ID (e.g., `dos-ai`)
- [ ] Test your application -- request/response format is identical
- [ ] (Optional) Update environment variable names for clarity:

```bash
# .env
# Before
OPENAI_API_KEY=sk-...

# After
DOS_AI_API_KEY=dos_sk_...
DOS_AI_BASE_URL=https://api.dos.ai/v1
```

## Using both OpenAI and DOS AI

You can use both providers in the same application by creating separate client instances:

```python
from openai import OpenAI

openai_client = OpenAI(api_key="sk-...")
dosai_client = OpenAI(base_url="https://api.dos.ai/v1", api_key="dos_sk_...")

# Route requests based on your needs
def get_completion(prompt, provider="dosai"):
    client = dosai_client if provider == "dosai" else openai_client
    model = "dos-ai" if provider == "dosai" else "gpt-4o"

    return client.chat.completions.create(
        model=model,
        messages=[{"role": "user", "content": prompt}],
    )
```

This makes it easy to evaluate DOS AI side-by-side or use it as a fallback provider.
