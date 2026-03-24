# Rate Limits

DOS AI enforces rate limits to ensure fair usage and platform stability for all users.

## Default Limits

| Metric | Limit |
| ------ | ----- |
| Requests per minute | 60 |
| Window type | Sliding window (60 seconds) |

The rate limiter uses a **sliding window** algorithm. This means the limit is calculated over a rolling 60-second period, not fixed calendar minutes.

## Rate Limit Headers

Every API response includes headers that report your current rate limit status:

| Header | Description |
| ------ | ----------- |
| `X-RateLimit-Limit` | Maximum requests allowed in the current window |
| `X-RateLimit-Remaining` | Requests remaining in the current window |

### Example Response Headers

```
HTTP/1.1 200 OK
X-RateLimit-Limit: 60
X-RateLimit-Remaining: 42
Content-Type: application/json
```

Use these headers to proactively manage your request rate and avoid hitting the limit.

## What Happens When You Hit the Limit

When you exceed the rate limit, the API returns a `429 Too Many Requests` response:

```json
{
  "error": {
    "message": "Rate limit exceeded. Please wait before making another request.",
    "type": "rate_limit_error",
    "code": 429
  }
}
```

The request is **not processed** and **no credits are charged**.

## Handling 429 Errors

### Exponential Backoff

The recommended strategy is **exponential backoff with jitter**. This progressively increases wait time between retries and adds randomness to prevent thundering herd problems.

#### Python Example

```python
import time
import random
import requests

def call_with_backoff(url, headers, data, max_retries=5):
    for attempt in range(max_retries):
        response = requests.post(url, headers=headers, json=data)

        if response.status_code == 200:
            return response.json()

        if response.status_code == 429:
            base_delay = 2 ** attempt  # 1, 2, 4, 8, 16 seconds
            jitter = random.uniform(0, base_delay * 0.5)
            wait_time = base_delay + jitter

            print(f"Rate limited. Retrying in {wait_time:.1f}s...")
            time.sleep(wait_time)
            continue

        response.raise_for_status()

    raise Exception("Max retries exceeded")
```

#### Node.js Example

```javascript
async function callWithBackoff(url, options, maxRetries = 5) {
  for (let attempt = 0; attempt < maxRetries; attempt++) {
    const response = await fetch(url, options);

    if (response.ok) {
      return await response.json();
    }

    if (response.status === 429) {
      const baseDelay = Math.pow(2, attempt) * 1000;
      const jitter = Math.random() * baseDelay * 0.5;
      const waitTime = baseDelay + jitter;

      console.log(`Rate limited. Retrying in ${(waitTime / 1000).toFixed(1)}s...`);
      await new Promise((resolve) => setTimeout(resolve, waitTime));
      continue;
    }

    throw new Error(`API error: ${response.status}`);
  }

  throw new Error("Max retries exceeded");
}
```

### Proactive Rate Management

Use the rate limit headers to throttle requests before hitting the limit:

```python
import time
import requests

def call_with_throttle(url, headers, data):
    response = requests.post(url, headers=headers, json=data)

    remaining = int(response.headers.get("X-RateLimit-Remaining", 60))

    if remaining < 5:
        time.sleep(2)
    elif remaining < 10:
        time.sleep(0.5)

    return response.json()
```

## Tips for Staying Within Limits

1. **Batch your work** -- Pace requests evenly rather than sending them all at once.
2. **Use streaming** -- Streaming responses (`stream: true`) count as a single request regardless of response length.
3. **Cache responses** -- Avoid making the same API call repeatedly.
4. **Monitor usage** -- Check `X-RateLimit-Remaining` headers and slow down as you approach the limit.
5. **Use fewer, larger requests** -- One comprehensive prompt is better than multiple small ones.

## Enterprise Custom Limits

For organizations with high-volume needs, we offer custom limits:

- **Higher request-per-minute caps** based on your workload
- **Burst allowances** for predictable traffic spikes
- **Dedicated capacity** with guaranteed throughput

Contact **support@dos.ai** to discuss custom rate limits for your organization.
