# OpenAPI Specification

The complete DOS AI API is documented using the OpenAPI Specification (OAS 3.1.0). You can access the machine-readable specification in either YAML or JSON format:

* **YAML:** [`https://api.dos.ai/openapi.yaml`](https://api.dos.ai/openapi.yaml)
* **JSON:** [`https://api.dos.ai/openapi.json`](https://api.dos.ai/openapi.json)

These specifications can be used with tools like **Swagger UI**, **Postman**, **Insomnia**, **Scalar**, or any OpenAPI-compatible SDK code generator to explore the API, run tests, or generate strongly typed client libraries.

---

## What is in the Specification?

The specification covers all core endpoints and capabilities of DOS AI:

| Tag | Key Endpoints | Description |
| --- | --- | --- |
| **Chat** | `POST /v1/chat/completions` | Chat completions, streaming SSE, tool calling, JSON schema mode, and per-request `cost` calculation. |
| **API Keys** | `GET /v1/key`, `GET /v1/auth/key` | API key metadata, usage, and balance checking (compatible with OpenRouter format). |
| **Billing & Balance** | `GET /v1/user/balance`, `GET /v1/credits` | Balance and credit queries (compatible with CC Switch, DeepSeek, and NewAPI formats). |
| **Models** | `GET /v1/models`, `GET /v1/models/{model_id}` | Dynamic model catalog with context windows and per-token `pricing`. |
| **Embeddings** | `POST /v1/embeddings` | Text embeddings generation. |
| **Anthropic Messages** | `POST /v1/messages`, `POST /v1/messages/count_tokens` | Anthropic-compatible Messages API and zero-cost token counting. |
| **Batches** | `POST /v1/files`, `POST /v1/batches`, `GET /v1/batches/{id}` | Asynchronous batch processing at 50% discount. |
| **Media Generation** | `POST /v1/{images,videos,audio}/generations` | Multimodal generation for images, videos, and music. |

---

## Using with Postman / Insomnia

You can import the complete DOS AI API collection into Postman or Insomnia with a single click:

1. Open **Postman** (or Insomnia).
2. Click **Import** in the top left corner.
3. Paste the URL:
   ```
   https://api.dos.ai/openapi.json
   ```
4. Click **Import**. Postman will generate an organized collection with all endpoints, headers, and request body templates.
5. Set your collection variable `apiKey` to your DOS AI API key (`dos_sk_...`) and start sending requests immediately.

---

## Using with Swagger UI / Scalar

To run an interactive Swagger UI locally with the DOS AI specification:

```bash
# Using Docker
docker run -p 8080:8080 -e SWAGGER_JSON_URL=https://api.dos.ai/openapi.json swaggerapi/swagger-ui
```

Visit `http://localhost:8080` in your browser to inspect and interact with the endpoints.

---

## Generating Client SDKs

You can use the official [OpenAPI Generator](https://openapi-generator.tech/) or modern tools like [Fern](https://buildwithfern.com/) or [Stainless](https://www.stainlessapi.com/) to automatically generate type-safe client libraries for your tech stack:

### Python Client

```bash
npx @openapitools/openapi-generator-cli generate \
  -i https://api.dos.ai/openapi.json \
  -g python \
  -o ./dos-ai-python-sdk \
  --additional-properties=packageName=dosai
```

### TypeScript / Node.js Client

```bash
npx @openapitools/openapi-generator-cli generate \
  -i https://api.dos.ai/openapi.json \
  -g typescript-axios \
  -o ./dos-ai-ts-sdk
```

### Go Client

```bash
npx @openapitools/openapi-generator-cli generate \
  -i https://api.dos.ai/openapi.json \
  -g go \
  -o ./dos-ai-go-sdk \
  --additional-properties=packageName=dosai
```

---

## Authentication in OpenAPI

All endpoints (except public routes such as `GET /v1/models`) require Bearer token authentication defined by the `BearerAuth` security scheme:

```http
Authorization: Bearer dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```
