# Function Calling

Function calling (also known as tool use) allows the model to invoke external functions you define. Instead of generating a text-only response, the model can output a structured function call with arguments, which your code executes and returns the result for the model to incorporate into its final answer.

This enables the model to interact with APIs, databases, calculators, and any external system.

## How It Works

1. You define one or more **tools** (functions) in the request, each with a name, description, and JSON Schema for parameters.
2. The model decides whether to call a tool based on the user's message.
3. If the model calls a tool, it returns a response with `finish_reason: "tool_calls"` containing the function name and arguments.
4. Your code executes the function and sends the result back.
5. The model generates a final response using the function result.

```
User message  -->  Model  -->  tool_calls response
                                    |
                          Your code executes function
                                    |
                          Tool result sent back  -->  Model  -->  Final response
```

## Defining Tools

Tools are defined in the `tools` array of the request. Each tool has a `type` of `"function"` and a `function` object containing the name, description, and parameter schema.

```json
{
  "model": "dos-ai",
  "messages": [
    { "role": "user", "content": "What is the weather in Hanoi?" }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a given city.",
        "parameters": {
          "type": "object",
          "properties": {
            "city": {
              "type": "string",
              "description": "The city name, e.g. 'Hanoi' or 'Ho Chi Minh City'."
            },
            "unit": {
              "type": "string",
              "enum": ["celsius", "fahrenheit"],
              "description": "Temperature unit. Defaults to celsius."
            }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

### Best Practices for Tool Definitions

- **Write clear descriptions.** The model uses the description to decide when to call the function. Be specific about what the function does and when it should be used.
- **Use JSON Schema constraints.** Use `enum`, `required`, `minimum`, `maximum`, and `pattern` to constrain parameters. This helps the model generate valid arguments.
- **Keep parameter names intuitive.** Use descriptive names like `city` rather than `c` or `param1`.

## Handling Tool Calls

When the model decides to call a tool, the response looks like this:

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"city\": \"Hanoi\", \"unit\": \"celsius\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

Key points:

- `content` may be `null` when the model makes a tool call.
- `arguments` is a JSON string that you need to parse.
- `id` on each tool call is required when sending the result back.
- `finish_reason` is `"tool_calls"` instead of `"stop"`.

## Sending Tool Results

After executing the function, send the result back by appending both the assistant's tool call message and a `tool` role message to the conversation:

```json
{
  "model": "dos-ai",
  "messages": [
    { "role": "user", "content": "What is the weather in Hanoi?" },
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [
        {
          "id": "call_abc123",
          "type": "function",
          "function": {
            "name": "get_weather",
            "arguments": "{\"city\": \"Hanoi\", \"unit\": \"celsius\"}"
          }
        }
      ]
    },
    {
      "role": "tool",
      "tool_call_id": "call_abc123",
      "content": "{\"temperature\": 32, \"unit\": \"celsius\", \"condition\": \"Partly cloudy\", \"humidity\": 75}"
    }
  ],
  "tools": [
    {
      "type": "function",
      "function": {
        "name": "get_weather",
        "description": "Get the current weather for a given city.",
        "parameters": {
          "type": "object",
          "properties": {
            "city": { "type": "string", "description": "The city name." },
            "unit": { "type": "string", "enum": ["celsius", "fahrenheit"] }
          },
          "required": ["city"]
        }
      }
    }
  ]
}
```

The model then generates a natural language response incorporating the tool result:

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": "The current weather in Hanoi is 32C and partly cloudy with 75% humidity."
      },
      "finish_reason": "stop"
    }
  ]
}
```

## Multiple Tool Calls

The model can call multiple tools in a single response. This is known as **parallel tool calling**.

```json
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_001",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"city\": \"Hanoi\"}"
            }
          },
          {
            "id": "call_002",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"city\": \"Ho Chi Minh City\"}"
            }
          }
        ]
      },
      "finish_reason": "tool_calls"
    }
  ]
}
```

When responding, include a separate `tool` message for each call, matched by `tool_call_id`:

```json
{
  "messages": [
    { "role": "user", "content": "Compare the weather in Hanoi and Ho Chi Minh City." },
    {
      "role": "assistant",
      "content": null,
      "tool_calls": [
        { "id": "call_001", "type": "function", "function": { "name": "get_weather", "arguments": "{\"city\": \"Hanoi\"}" } },
        { "id": "call_002", "type": "function", "function": { "name": "get_weather", "arguments": "{\"city\": \"Ho Chi Minh City\"}" } }
      ]
    },
    {
      "role": "tool",
      "tool_call_id": "call_001",
      "content": "{\"temperature\": 32, \"condition\": \"Partly cloudy\"}"
    },
    {
      "role": "tool",
      "tool_call_id": "call_002",
      "content": "{\"temperature\": 35, \"condition\": \"Sunny\"}"
    }
  ]
}
```

## Controlling Tool Use

You can control whether and how the model uses tools with the `tool_choice` parameter:

| Value | Behavior |
|-------|----------|
| `"auto"` | The model decides whether to call a tool (default). |
| `"none"` | The model will not call any tools, even if they are defined. |
| `"required"` | The model must call at least one tool. |
| `{"type": "function", "function": {"name": "get_weather"}}` | Force the model to call a specific function. |

```json
{
  "model": "dos-ai",
  "messages": [{ "role": "user", "content": "What is the weather?" }],
  "tools": ["..."],
  "tool_choice": "required"
}
```

## Complete Example

### Python

```python
import json
from openai import OpenAI

client = OpenAI(
    api_key="dos_sk_your_api_key_here",
    base_url="https://api.dos.ai/v1",
)

# Define available tools
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "Get the current weather for a city.",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "The city name.",
                    },
                    "unit": {
                        "type": "string",
                        "enum": ["celsius", "fahrenheit"],
                        "description": "Temperature unit.",
                    },
                },
                "required": ["city"],
            },
        },
    },
]


# Simulate the actual function
def get_weather(city: str, unit: str = "celsius") -> dict:
    # In production, call a real weather API
    return {
        "city": city,
        "temperature": 32,
        "unit": unit,
        "condition": "Partly cloudy",
    }


# Step 1: Send the user message with tools
messages = [{"role": "user", "content": "What is the weather like in Hanoi?"}]

response = client.chat.completions.create(
    model="dos-ai",
    messages=messages,
    tools=tools,
)

assistant_message = response.choices[0].message

# Step 2: Check if the model wants to call a tool
if assistant_message.tool_calls:
    # Append the assistant message (with tool calls) to history
    messages.append(assistant_message)

    # Execute each tool call
    for tool_call in assistant_message.tool_calls:
        function_name = tool_call.function.name
        arguments = json.loads(tool_call.function.arguments)

        # Dispatch to the actual function
        if function_name == "get_weather":
            result = get_weather(**arguments)
        else:
            result = {"error": f"Unknown function: {function_name}"}

        # Append the tool result
        messages.append(
            {
                "role": "tool",
                "tool_call_id": tool_call.id,
                "content": json.dumps(result),
            }
        )

    # Step 3: Get the final response with tool results
    final_response = client.chat.completions.create(
        model="dos-ai",
        messages=messages,
        tools=tools,
    )

    print(final_response.choices[0].message.content)
else:
    # No tool call -- direct response
    print(assistant_message.content)
```

### JavaScript

```javascript
import OpenAI from "openai";

const client = new OpenAI({
  apiKey: "dos_sk_your_api_key_here",
  baseURL: "https://api.dos.ai/v1",
});

const tools = [
  {
    type: "function",
    function: {
      name: "get_weather",
      description: "Get the current weather for a city.",
      parameters: {
        type: "object",
        properties: {
          city: { type: "string", description: "The city name." },
          unit: {
            type: "string",
            enum: ["celsius", "fahrenheit"],
            description: "Temperature unit.",
          },
        },
        required: ["city"],
      },
    },
  },
];

// Simulate the actual function
function getWeather(city, unit = "celsius") {
  return { city, temperature: 32, unit, condition: "Partly cloudy" };
}

// Step 1: Send user message with tools
const messages = [
  { role: "user", content: "What is the weather like in Hanoi?" },
];

let response = await client.chat.completions.create({
  model: "dos-ai",
  messages,
  tools,
});

const assistantMessage = response.choices[0].message;

// Step 2: Check if the model wants to call a tool
if (assistantMessage.tool_calls) {
  messages.push(assistantMessage);

  for (const toolCall of assistantMessage.tool_calls) {
    const args = JSON.parse(toolCall.function.arguments);

    let result;
    if (toolCall.function.name === "get_weather") {
      result = getWeather(args.city, args.unit);
    } else {
      result = { error: `Unknown function: ${toolCall.function.name}` };
    }

    messages.push({
      role: "tool",
      tool_call_id: toolCall.id,
      content: JSON.stringify(result),
    });
  }

  // Step 3: Get the final response
  response = await client.chat.completions.create({
    model: "dos-ai",
    messages,
    tools,
  });

  console.log(response.choices[0].message.content);
} else {
  console.log(assistantMessage.content);
}
```

### cURL

```bash
# Step 1: Send user message with tools
curl https://api.dos.ai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dos_sk_your_api_key_here" \
  -d '{
    "model": "dos-ai",
    "messages": [
      {"role": "user", "content": "What is the weather in Hanoi?"}
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_weather",
          "description": "Get the current weather for a city.",
          "parameters": {
            "type": "object",
            "properties": {
              "city": {"type": "string", "description": "The city name."},
              "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["city"]
          }
        }
      }
    ]
  }'

# Step 2: After parsing the tool call response, send back the result
curl https://api.dos.ai/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer dos_sk_your_api_key_here" \
  -d '{
    "model": "dos-ai",
    "messages": [
      {"role": "user", "content": "What is the weather in Hanoi?"},
      {
        "role": "assistant",
        "content": null,
        "tool_calls": [{
          "id": "call_abc123",
          "type": "function",
          "function": {
            "name": "get_weather",
            "arguments": "{\"city\": \"Hanoi\", \"unit\": \"celsius\"}"
          }
        }]
      },
      {
        "role": "tool",
        "tool_call_id": "call_abc123",
        "content": "{\"temperature\": 32, \"unit\": \"celsius\", \"condition\": \"Partly cloudy\"}"
      }
    ],
    "tools": [
      {
        "type": "function",
        "function": {
          "name": "get_weather",
          "description": "Get the current weather for a city.",
          "parameters": {
            "type": "object",
            "properties": {
              "city": {"type": "string", "description": "The city name."},
              "unit": {"type": "string", "enum": ["celsius", "fahrenheit"]}
            },
            "required": ["city"]
          }
        }
      }
    ]
  }'
```

## Error Handling

Common issues with function calling:

| Issue | Cause | Solution |
|-------|-------|----------|
| Model ignores tools | Description is too vague | Write a clearer, more specific function description. |
| Invalid arguments | Schema is too loose | Add `required`, `enum`, and constraints to the schema. |
| Model hallucinates functions | Too many tools defined | Reduce the number of tools, or use `tool_choice` to constrain. |
| JSON parse error on `arguments` | Model output was malformed | Wrap `JSON.parse` in a try/catch and retry the request. |

## Next Steps

- [Chat Completions](chat-completions.md) -- learn the basics of the API.
- [Streaming](streaming.md) -- stream tool call responses in real time.
- [Structured Outputs](structured-outputs.md) -- get reliable JSON from the model.
