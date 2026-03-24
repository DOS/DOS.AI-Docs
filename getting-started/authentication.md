# Authentication

Every request to the DOS AI API must include a valid API key. This page covers how to create, use, and secure your keys.

## API key format

DOS AI keys follow the format:

```
dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

All keys begin with the `dos_sk_` prefix. Keys are hashed (SHA-256) before storage -- we never store your raw key. If you lose a key, you'll need to generate a new one.

## Creating an API key

1. Log in to the [DOS AI dashboard](https://app.dos.ai).
2. Go to **API Keys** in the sidebar.
3. Click **Create new key**.
4. Give your key a descriptive name (e.g., "Production backend", "Local development").
5. Copy the key immediately. It will only be displayed once.

You can create multiple keys to separate concerns -- for example, one key per environment or per team member.

## Using your API key

Include your key in the `Authorization` header of every request:

```
Authorization: Bearer dos_sk_...
```

### With the OpenAI SDK (Python)

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.dos.ai/v1",
    api_key="dos_sk_...",
)
```

### With the OpenAI SDK (JavaScript)

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://api.dos.ai/v1",
  apiKey: "dos_sk_...",
});
```

### With cURL

```bash
curl https://api.dos.ai/v1/chat/completions \
  -H "Authorization: Bearer dos_sk_..." \
  -H "Content-Type: application/json" \
  -d '{"model": "dos-ai", "messages": [{"role": "user", "content": "Hello"}]}'
```

### With environment variables (recommended)

Rather than hardcoding your key, set it as an environment variable:

```bash
export DOS_AI_API_KEY="dos_sk_..."
```

Then read it in your code:

**Python:**

```python
import os
from openai import OpenAI

client = OpenAI(
    base_url="https://api.dos.ai/v1",
    api_key=os.environ["DOS_AI_API_KEY"],
)
```

**JavaScript:**

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://api.dos.ai/v1",
  apiKey: process.env.DOS_AI_API_KEY,
});
```

## Managing keys

From the dashboard you can:

- **View all keys** -- See key names, creation dates, and last-used timestamps.
- **Delete a key** -- Immediately revokes access. Any request using that key will return `401 Unauthorized`.
- **Create new keys** -- No limit on the number of active keys per account.

## Rate limits

Rate limits are applied per API key using a sliding window (60-second window).

| Plan | Requests per minute | Tokens per minute |
| --- | --- | --- |
| Free | 60 | 100,000 |
| Plus | 120 | 500,000 |
| Pro | 300 | 2,000,000 |

When you exceed a rate limit, the API returns HTTP `429 Too Many Requests` with a `Retry-After` header indicating how long to wait.

## Error responses

| Status code | Meaning |
| --- | --- |
| `401 Unauthorized` | Missing or invalid API key |
| `402 Payment Required` | Insufficient credit balance |
| `429 Too Many Requests` | Rate limit exceeded |

Example error response:

```json
{
  "error": {
    "message": "Invalid API key",
    "type": "authentication_error",
    "code": "invalid_api_key"
  }
}
```

## Security best practices

1. **Never commit keys to version control.** Use environment variables or a secrets manager (e.g., Doppler, AWS Secrets Manager, Vercel Environment Variables).

2. **Use separate keys for each environment.** Create distinct keys for development, staging, and production. If a dev key leaks, revoke it without affecting production.

3. **Rotate keys periodically.** Generate a new key, update your deployment, then delete the old key.

4. **Restrict access.** Only share keys with team members who need them. Use your organization's secrets management tooling.

5. **Monitor usage.** Check the dashboard regularly for unexpected spikes that could indicate a leaked key.

If you believe a key has been compromised, delete it immediately from the dashboard and create a new one.
