# API Key & Balance

Endpoints to inspect API key metadata, check spending limits, and query current account credit balance. These endpoints are designed to be 100% drop-in compatible with **OpenRouter**, **DeepSeek**, **CC Switch**, **Cline**, **Roo Code**, and **OpenWebUI**.

---

## 1. Check API Key & Quota (`GET /v1/key`)

Inspects the status, friendly label, usage, and remaining balance associated with the caller's API key. This follows the official [OpenRouter Key API specification](https://openrouter.ai/docs/api_reference/overview).

### Endpoint

```http
GET https://api.dos.ai/v1/key
GET https://api.dos.ai/v1/auth/key
```

> **Root Alias:** If your client configures the base URL as `https://api.dos.ai` (without `/v1`), requests to `GET /key` and `GET /auth/key` are also supported.

### Authentication

```http
Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Response

#### Success (200 OK)

```json
{
  "data": {
    "label": "My Project Key",
    "usage": 14.25,
    "limit": null,
    "is_free_tier": false,
    "rate_limit": {
      "requests": 500,
      "interval": "10s"
    },
    "limit_remaining": 35.75
  }
}
```

### Response Fields

| Field | Type | Description |
| --- | --- | --- |
| `data.label` | string | The friendly name or label given to this API key in the dashboard. |
| `data.usage` | number | Total amount spent in USD by this account. |
| `data.limit` | number \| null | Hard spending limit set for this key/account in USD. Returns `null` for pay-as-you-go accounts without an explicit spending cap. |
| `data.limit_remaining` | number | Current available credit balance in USD that can be used for requests. |
| `data.is_free_tier` | boolean | Set to `true` if the account is on an initial free tier or if balance has been exhausted. |
| `data.rate_limit` | object | The rate limit enforced for this key. |
| `data.rate_limit.requests` | integer | Number of allowed requests per interval window. |
| `data.rate_limit.interval` | string | The duration of the rate limit window (e.g., `"10s"`). |

---

## 2. Query User Balance (`GET /v1/user/balance`)

Queries account balance and credit status. Designed for full compatibility with **CC Switch**, **DeepSeek User Balance API**, and **NewAPI / OneAPI**.

### Endpoint

```http
GET https://api.dos.ai/v1/user/balance
POST https://api.dos.ai/v1/user/balance
GET https://api.dos.ai/v1/credits
```

> **Root Alias:** Requests to `GET /user/balance`, `POST /user/balance`, and `GET /credits` directly on `https://api.dos.ai` are also supported.

### Authentication

```http
Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

### Response

#### Active Account with Balance (200 OK)

```json
{
  "is_active": true,
  "is_available": true,
  "invalid_message": "",
  "balance": 35.75,
  "currency": "USD",
  "unit": "USD",
  "total_deposited": 50.00,
  "total_spent": 14.25,
  "plan": "pay_as_you_go",
  "user_id": "usr_abc123xyz",
  "data": {
    "total_credits": 50.00,
    "total_usage": 14.25,
    "remaining_credits": 35.75
  },
  "balance_infos": [
    {
      "currency": "USD",
      "total_balance": "35.75",
      "granted_balance": "0.00",
      "topped_up_balance": "35.75"
    }
  ]
}
```

#### Exhausted Balance (200 OK)

When an account has no remaining balance (`balance <= 0`):

```json
{
  "is_active": false,
  "is_available": false,
  "invalid_message": "Credit balance exhausted. Top up at https://app.dos.ai/billing",
  "balance": 0.00,
  "currency": "USD",
  "unit": "USD",
  "total_deposited": 50.00,
  "total_spent": 50.00,
  "plan": "pay_as_you_go",
  "user_id": "usr_abc123xyz",
  "data": {
    "total_credits": 50.00,
    "total_usage": 50.00,
    "remaining_credits": 0.00
  },
  "balance_infos": [
    {
      "currency": "USD",
      "total_balance": "0.00",
      "granted_balance": "0.00",
      "topped_up_balance": "0.00"
    }
  ]
}
```

---

## Examples

### cURL

```bash
# Check key via OpenRouter format
curl https://api.dos.ai/v1/key \
  -H "Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# Query balance via DeepSeek / CC Switch format
curl https://api.dos.ai/v1/user/balance \
  -H "Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

### Python

```python
import requests

api_key = "dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
headers = {"Authorization": f"Bearer {api_key}"}

# Check key status
key_info = requests.get("https://api.dos.ai/v1/key", headers=headers).json()
print(f"Key Label: {key_info['data']['label']}")
print(f"Remaining: ${key_info['data']['limit_remaining']:.2f}")

# Check balance
balance_info = requests.get("https://api.dos.ai/v1/user/balance", headers=headers).json()
print(f"Active: {balance_info['is_active']}")
print(f"Balance: ${balance_info['balance']:.2f} {balance_info['currency']}")
```

### JavaScript / Node.js

```javascript
const apiKey = "dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";

const res = await fetch("https://api.dos.ai/v1/key", {
  headers: {
    "Authorization": `Bearer ${apiKey}`
  }
});
const { data } = await res.json();
console.log(`Key: ${data.label}, Remaining: $${data.limit_remaining}`);
```
