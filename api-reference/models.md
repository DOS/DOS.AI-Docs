# Models

Retrieve the list of models currently available on DOS AI.

## Endpoint

```
GET https://api.dos.ai/v1/models
```

## Authentication

Include your API key in the `Authorization` header:

```
Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

## Response

Returns a JSON object containing a `data` array of model objects.

| Field | Type | Description |
| ----- | ---- | ----------- |
| `object` | string | Always `"list"` |
| `data` | array | Array of model objects |

### Model object

| Field | Type | Description |
| ----- | ---- | ----------- |
| `id` | string | The model identifier used in API requests (e.g. `dos-ai`) |
| `object` | string | Always `"model"` |
| `created` | integer | Unix timestamp of when the model was registered |
| `owned_by` | string | The organization that owns the model |

### Example response

```json
{
  "object": "list",
  "data": [
    {
      "id": "dos-ai",
      "object": "model",
      "created": 1700000000,
      "owned_by": "dos-ai"
    }
  ]
}
```

## Examples

### cURL

```bash
curl https://api.dos.ai/v1/models \
  -H "Authorization: Bearer dos_sk_..."
```

### Python

```python
from openai import OpenAI

client = OpenAI(
    base_url="https://api.dos.ai/v1",
    api_key="dos_sk_...",
)

models = client.models.list()

for model in models.data:
    print(model.id)
```

### JavaScript

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  baseURL: "https://api.dos.ai/v1",
  apiKey: "dos_sk_...",
});

const models = await client.models.list();

for (const model of models.data) {
  console.log(model.id);
}
```

## Notes

- The models endpoint is OpenAI-compatible. Any client or library that supports the OpenAI `/v1/models` endpoint will work without modification.
- The list reflects models currently loaded and ready to serve requests. Available models may change over time -- see [Available Models](../models/available-models.md) for the full catalog and pricing.
