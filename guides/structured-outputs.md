# Structured Outputs

Structured outputs let you constrain the model to return valid JSON that conforms to a specific schema. This is essential when the model's response needs to be parsed by code rather than read by a human.

## JSON Mode

The simplest way to get JSON output is to set `response_format` to `{ "type": "json_object" }`. This guarantees the model's response is valid JSON.

```json
{
  "model": "dos-ai",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful assistant that always responds in JSON format."
    },
    {
      "role": "user",
      "content": "List 3 programming languages with their year of creation."
    }
  ],
  "response_format": { "type": "json_object" }
}
```

Response:

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "{\n  \"languages\": [\n    {\"name\": \"Python\", \"year\": 1991},\n    {\"name\": \"JavaScript\", \"year\": 1995},\n    {\"name\": \"Go\", \"year\": 2009}\n  ]\n}"
      },
      "finish_reason": "stop"
    }
  ]
}
```

The `content` field is a JSON string. Parse it in your code:

```python
import json

data = json.loads(response.choices[0].message.content)
print(data["languages"][0]["name"])  # "Python"
```

> **Important:** When using JSON mode, you must include the word "JSON" somewhere in the system or user message. This is a safety measure to ensure the user intends to receive JSON output.

## JSON Schema Enforcement

For stronger guarantees, you can provide a JSON Schema that the model's output must conform to. This uses `response_format` with `type: "json_schema"`:

```json
{
  "model": "dos-ai",
  "messages": [
    {
      "role": "user",
      "content": "Analyze the sentiment of this review: 'The product arrived quickly and works perfectly. Very satisfied with the purchase.'"
    }
  ],
  "response_format": {
    "type": "json_schema",
    "json_schema": {
      "name": "sentiment_analysis",
      "strict": true,
      "schema": {
        "type": "object",
        "properties": {
          "sentiment": {
            "type": "string",
            "enum": ["positive", "negative", "neutral", "mixed"]
          },
          "confidence": {
            "type": "number",
            "description": "Confidence score between 0 and 1."
          },
          "summary": {
            "type": "string",
            "description": "Brief explanation of the sentiment."
          },
          "keywords": {
            "type": "array",
            "items": { "type": "string" },
            "description": "Key phrases that influenced the analysis."
          }
        },
        "required": ["sentiment", "confidence", "summary", "keywords"],
        "additionalProperties": false
      }
    }
  }
}
```

Response:

```json
{
  "choices": [
    {
      "message": {
        "content": "{\"sentiment\": \"positive\", \"confidence\": 0.95, \"summary\": \"The reviewer expresses strong satisfaction with both shipping speed and product quality.\", \"keywords\": [\"arrived quickly\", \"works perfectly\", \"very satisfied\"]}"
      }
    }
  ]
}
```

### Schema Rules

When using `strict: true`, the following rules apply:

- All fields must be listed in `required`.
- `additionalProperties` must be set to `false`.
- Supported types: `string`, `number`, `integer`, `boolean`, `array`, `object`, `null`.
- Use `enum` to restrict string values.
- Nested objects must also follow these rules.

## Code Examples

### Python

```python
import json
from openai import OpenAI

client = OpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
)

# Simple JSON mode
response = client.chat.completions.create(
    model="dos-ai",
    messages=[
        {
            "role": "system",
            "content": "Extract the entities from the text. Respond in JSON with keys: persons (array of strings), locations (array of strings), organizations (array of strings).",
        },
        {
            "role": "user",
            "content": "Elon Musk announced that Tesla will open a new factory in Berlin, in partnership with the German government.",
        },
    ],
    response_format={"type": "json_object"},
)

data = json.loads(response.choices[0].message.content)
print(data)
# {
#   "persons": ["Elon Musk"],
#   "locations": ["Berlin"],
#   "organizations": ["Tesla", "German government"]
# }
```

### Python with JSON Schema

```python
import json
from openai import OpenAI

client = OpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
)

response = client.chat.completions.create(
    model="dos-ai",
    messages=[
        {
            "role": "user",
            "content": "Generate a recipe for pho bo (Vietnamese beef noodle soup).",
        }
    ],
    response_format={
        "type": "json_schema",
        "json_schema": {
            "name": "recipe",
            "strict": True,
            "schema": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "servings": {"type": "integer"},
                    "prep_time_minutes": {"type": "integer"},
                    "cook_time_minutes": {"type": "integer"},
                    "ingredients": {
                        "type": "array",
                        "items": {
                            "type": "object",
                            "properties": {
                                "item": {"type": "string"},
                                "quantity": {"type": "string"},
                            },
                            "required": ["item", "quantity"],
                            "additionalProperties": False,
                        },
                    },
                    "steps": {
                        "type": "array",
                        "items": {"type": "string"},
                    },
                },
                "required": [
                    "name",
                    "servings",
                    "prep_time_minutes",
                    "cook_time_minutes",
                    "ingredients",
                    "steps",
                ],
                "additionalProperties": False,
            },
        },
    },
)

recipe = json.loads(response.choices[0].message.content)
print(f"Recipe: {recipe['name']}")
print(f"Servings: {recipe['servings']}")
print(f"Prep: {recipe['prep_time_minutes']} min, Cook: {recipe['cook_time_minutes']} min")
for ingredient in recipe["ingredients"]:
    print(f"  - {ingredient['quantity']} {ingredient['item']}")
```

### JavaScript

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "dos_sk_your_api_key_here",
  baseURL: "https://api.dos.ai/v1",
});

// Simple JSON mode
const response = await client.chat.completions.create({
  model: "dos-ai",
  messages: [
    {
      role: "system",
      content:
        "Extract entities from the text. Respond in JSON with keys: persons, locations, organizations (all arrays of strings).",
    },
    {
      role: "user",
      content:
        "Elon Musk announced that Tesla will open a new factory in Berlin.",
    },
  ],
  response_format: { type: "json_object" },
});

const data = JSON.parse(response.choices[0].message.content);
console.log(data);
```

### JavaScript with JSON Schema

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "dos_sk_your_api_key_here",
  baseURL: "https://api.dos.ai/v1",
});

const response = await client.chat.completions.create({
  model: "dos-ai",
  messages: [
    {
      role: "user",
      content:
        "Classify this support ticket: 'My order #12345 has not arrived after 2 weeks. I want a refund.'",
    },
  ],
  response_format: {
    type: "json_schema",
    json_schema: {
      name: "ticket_classification",
      strict: true,
      schema: {
        type: "object",
        properties: {
          category: {
            type: "string",
            enum: ["billing", "shipping", "product", "account", "other"],
          },
          priority: {
            type: "string",
            enum: ["low", "medium", "high", "urgent"],
          },
          summary: { type: "string" },
          requires_human: { type: "boolean" },
        },
        required: ["category", "priority", "summary", "requires_human"],
        additionalProperties: false,
      },
    },
  },
});

const ticket = JSON.parse(response.choices[0].message.content);
console.log(ticket);
// {
//   category: "shipping",
//   priority: "high",
//   summary: "Customer reports undelivered order after 2 weeks and requests refund.",
//   requires_human: true
// }
```

### cURL

```bash
curl https://api.dos.ai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dos_sk_your_api_key_here" \
  -d '{
    "model": "dos-ai",
    "messages": [
      {
        "role": "system",
        "content": "You are a data extraction assistant. Respond in JSON."
      },
      {
        "role": "user",
        "content": "Extract contact info: John Smith, john@example.com, +1-555-0123, works at Acme Corp."
      }
    ],
    "response_format": {"type": "json_object"}
  }'
```

## Tips for Reliable JSON Output

### 1. Always mention JSON in the prompt

When using `response_format: { type: "json_object" }`, include an explicit instruction about JSON in the system message. Without it, the model may produce an error or unpredictable output.

```python
# Good
messages = [
    {"role": "system", "content": "Respond with a JSON object containing..."},
    {"role": "user", "content": "..."}
]

# Bad -- may fail without JSON mentioned in messages
messages = [
    {"role": "user", "content": "List some countries."}
]
```

### 2. Describe the expected shape

Even with JSON mode, the model needs to know what structure you expect. Be explicit about field names and types:

```
Respond with a JSON object with these fields:
- "name" (string): the item name
- "price" (number): price in USD
- "in_stock" (boolean): availability status
- "tags" (array of strings): relevant categories
```

### 3. Use JSON Schema for critical applications

For production systems where invalid JSON would cause failures, prefer `json_schema` with `strict: true` over plain `json_object` mode. The schema enforcement happens at the decoding level, guaranteeing structural correctness.

### 4. Handle parsing errors gracefully

Even with structured outputs, always wrap JSON parsing in error handling:

```python
import json

try:
    data = json.loads(response.choices[0].message.content)
except json.JSONDecodeError as e:
    print(f"Failed to parse JSON: {e}")
    # Retry or fall back to a default
```

### 5. Keep schemas simple

Deeply nested schemas with many optional fields increase the chance of unexpected output. Flatten where possible and use `required` to ensure critical fields are always present.

## Comparison: JSON Mode vs JSON Schema

| Feature | JSON Mode | JSON Schema |
|---------|-----------|-------------|
| Setting | `{"type": "json_object"}` | `{"type": "json_schema", "json_schema": {...}}` |
| Guarantee | Valid JSON | Valid JSON conforming to your schema |
| Field control | Must be described in prompt | Enforced by schema |
| `strict` mode | Not applicable | Optional (recommended) |
| Use case | Simple/flexible JSON responses | Production APIs, data pipelines |

## Next Steps

- [Chat Completions](chat-completions.md) -- learn the basics of the API.
- [Function Calling](function-calling.md) -- let the model call external tools.
- [Streaming](streaming.md) -- receive tokens in real time.
