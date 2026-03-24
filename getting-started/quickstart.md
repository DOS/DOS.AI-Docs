# Quickstart

Go from zero to your first AI-generated response in under five minutes.

## 1. Create an account

Sign up at [app.dos.ai](https://app.dos.ai). Every new account receives **$5 in free credits** -- no credit card required.

## 2. Get your API key

1. Log in to the [DOS AI dashboard](https://app.dos.ai).
2. Navigate to **API Keys** in the sidebar.
3. Click **Create new key**.
4. Copy the key (it starts with `dos_sk_`). Store it somewhere safe -- you won't be able to see it again.

## 3. Make your first request

DOS AI is fully compatible with the OpenAI SDK. Install it and point it at our API.

### Python

```bash
pip install openai
```

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.dos.ai/v1",
    api_key="dos_sk_...",  # Replace with your key
)

response = client.chat.completions.create(
    model="dos-ai",
    messages=[
        {"role": "user", "content": "What is the capital of France?"}
    ],
)

print(response.choices[0].message.content)
```

### JavaScript / TypeScript

```bash
npm install openai
```

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://api.dos.ai/v1",
  apiKey: "dos_sk_...", // Replace with your key
});

const response = await client.chat.completions.create({
  model: "dos-ai",
  messages: [
    { role: "user", content: "What is the capital of France?" }
  ],
});

console.log(response.choices[0].message.content);
```

### cURL

```bash
curl https://api.dos.ai/v1/chat/completions \
  -H "Authorization: Bearer dos_sk_..." \
  -H "Content-Type: application/json" \
  -d '{
    "model": "dos-ai",
    "messages": [
      {"role": "user", "content": "What is the capital of France?"}
    ]
  }'
```

## 4. Try streaming

Streaming lets you receive tokens as they are generated, giving your users a real-time typing experience.

### Python (streaming)

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.dos.ai/v1",
    api_key="dos_sk_...",
)

stream = client.chat.completions.create(
    model="dos-ai",
    messages=[
        {"role": "user", "content": "Write a short poem about open-source AI."}
    ],
    stream=True,
)

for chunk in stream:
    content = chunk.choices[0].delta.content
    if content:
        print(content, end="", flush=True)
print()
```

### JavaScript (streaming)

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://api.dos.ai/v1",
  apiKey: "dos_sk_...",
});

const stream = await client.chat.completions.create({
  model: "dos-ai",
  messages: [
    { role: "user", content: "Write a short poem about open-source AI." }
  ],
  stream: true,
});

for await (const chunk of stream) {
  const content = chunk.choices[0]?.delta?.content;
  if (content) process.stdout.write(content);
}
console.log();
```

### cURL (streaming)

```bash
curl https://api.dos.ai/v1/chat/completions \
  -H "Authorization: Bearer dos_sk_..." \
  -H "Content-Type: application/json" \
  -d '{
    "model": "dos-ai",
    "messages": [
      {"role": "user", "content": "Write a short poem about open-source AI."}
    ],
    "stream": true
  }'
```

## 5. Explore further

- **Manage your keys**: [Authentication](authentication.md)
- **Migrate from OpenAI**: [OpenAI Compatibility](openai-compatibility.md)
- **Check available models**: `GET https://api.dos.ai/v1/models`

> **Tip:** Since DOS AI is OpenAI-compatible, any tutorial, library, or framework that works with the OpenAI API also works with DOS AI. Just change the base URL and API key.
