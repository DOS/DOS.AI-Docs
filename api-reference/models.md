# List Models

Retrieve the list of models currently available on the DOS AI platform. This endpoint is useful for dynamically discovering which models you can use without hard-coding model IDs.

## Endpoint

```
GET https://api.dos.ai/v1/models
```

## Authentication

Include your API key in the `Authorization` header:

```
Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Request

No request body or query parameters are required.

## Response

### Success (200 OK)

```json
{
  "object": "list",
  "data": [
    {
      "id": "dos-ai",
      "object": "model",
      "created": 1711000000,
      "owned_by": "dos-ai"
    },
    {
      "id": "llama-3.3-70b",
      "object": "model",
      "created": 1711000000,
      "owned_by": "dos-ai"
    },
    {
      "id": "deepseek-v3",
      "object": "model",
      "created": 1711000000,
      "owned_by": "dos-ai"
    },
    {
      "id": "llama-3.1-8b",
      "object": "model",
      "created": 1711000000,
      "owned_by": "dos-ai"
    }
  ]
}
```

### Response Fields

| Field | Type | Description |
| ----- | ---- | ----------- |
| `object` | string | Always `"list"`. |
| `data` | array | Array of model objects. |
| `data[].id` | string | The model identifier. Use this as the `model` parameter in API requests. |
| `data[].object` | string | Always `"model"`. |
| `data[].created` | integer | Unix timestamp of when the model was added. |
| `data[].owned_by` | string | The organization that owns the model. |

## Model ID Mapping

The `id` field is the value you use when specifying a model in API requests:

| Model Name | Model ID |
| ---------- | -------- |
| Qwen3.5-35B-A3B | `dos-ai` |
| Llama 3.3 70B | `llama-3.3-70b` |
| DeepSeek V3 | `deepseek-v3` |
| Llama 3.1 8B | `llama-3.1-8b` |

## Error Responses

| Status Code | Description |
| ----------- | ----------- |
| 401 | Invalid or missing API key. |
| 500 | Internal server error. |

See [Error Codes](../support/error-codes.md) for details.

## Examples

### cURL

```bash
curl https://api.dos.ai/v1/models \
  -H "Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

### Python

```python
from openai import OpenAI

client = OpenAI(
    api_key="dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    base_url="https://api.dos.ai/v1"
)

models = client.models.list()
for model in models.data:
    print(model.id)
```

### Node.js

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
  baseURL: "https://api.dos.ai/v1",
});

const models = await client.models.list();
for (const model of models.data) {
  console.log(model.id);
}
```

> **Note:** The list of available models may change over time as new models are added. We recommend fetching the model list dynamically rather than hard-coding model IDs in your application.
