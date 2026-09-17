# Batches (Batch API)

The Batch API lets you submit asynchronous, high-volume workloads at a **50% discount** off standard retail rates. Instead of executing requests sequentially and waiting synchronously for each response, you upload a JSONL file of requests, create a batch with a 24-hour completion window, and download the results once processing completes.

The DOS.AI Batch API is **100% OpenAI-compatible**, so you can use existing OpenAI SDKs or HTTP clients without modifying your code structure.

---

## Why Use the Batch API?

- **50% Cost Savings**: All batch requests are charged at exactly 50% of the standard retail token price.
- **Dual-Engine Processing**:
  - **Self-Hosted (`dos`)**: Processed during local GPU idle capacity windows.
  - **Cloud Relay (Alibaba Cloud DeepSeek & Qwen)**: Scheduled during provider off-peak windows (**00:00 – 10:00 GMT+8**), giving you reliable, low-cost processing for frontier open-weights models.
- **Independent Limits**: Batch volume does not consume your real-time interactive rate limits.
- **Resilient & Fault-Tolerant**: Individual request failures (e.g. invalid formatting or context length exceedance) are recorded per line in the output file without failing the rest of the batch.

---

## Supported Models

| Model ID | Provider | Description | Discount |
| :--- | :--- | :--- | :---: |
| `dos` (or `dos-ai`) | Self-Hosted | DOS.AI's flagship self-hosted model | **-50%** |
| `deepseek-v4-pro` | Alibaba Cloud | Frontier reasoning and coding model | **-50%** |
| `deepseek-v4.1-flash` | Alibaba Cloud | Ultra-fast, cost-effective general intelligence | **-50%** |
| `deepseek-v4` | Alibaba Cloud | Flagship DeepSeek V4 foundation model | **-50%** |
| `deepseek-chat` | Alibaba Cloud | Standard DeepSeek conversational model | **-50%** |
| `qwen3.8-27b` | Alibaba Cloud | Qwen 3.8 27B high-efficiency instruct model | **-50%** |
| `qwen3.8-max` | Alibaba Cloud | Qwen 3.8 flagship frontier model | **-50%** |
| `qwen3.8-72b` | Alibaba Cloud | Qwen 3.8 dense open-weights model | **-50%** |

---

## Endpoints

| Endpoint | Method | Description |
| :--- | :--- | :--- |
| `https://api.dos.ai/v1/files` | `POST` | Upload a JSONL input file (`purpose="batch"`). |
| `https://api.dos.ai/v1/batches` | `POST` | Create a batch job referencing the uploaded file. |
| `https://api.dos.ai/v1/batches/{batch_id}` | `GET` | Retrieve the status and request counts of a batch. |
| `https://api.dos.ai/v1/files/{file_id}/content` | `GET` | Download the results JSONL content. |
| `https://api.dos.ai/v1/batches/{batch_id}/cancel` | `POST` | Cancel an in-flight or validating batch job. |

---

## Step-by-Step Walkthrough

### 1. Prepare Your Input JSONL File

Each line in the file must be a JSON object containing `custom_id`, `method`, `url`, and `body`:

```jsonl
{"custom_id": "req-1", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "deepseek-v4-pro", "messages": [{"role": "user", "content": "Explain quantum computing in 3 bullet points."}]}}
{"custom_id": "req-2", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "deepseek-v4-pro", "messages": [{"role": "user", "content": "Write a Python script to calculate Fibonacci numbers."}]}}
{"custom_id": "req-3", "method": "POST", "url": "/v1/chat/completions", "body": {"model": "dos", "messages": [{"role": "user", "content": "Summarize the history of AI in 50 words."}]}}
```

Save this file as `batch_input.jsonl`.

### 2. Upload the File (`POST /v1/files`)

Upload your JSONL file with `purpose="batch"`:

```bash
curl https://api.dos.ai/v1/files \
  -H "Authorization: Bearer $DOS_API_KEY" \
  -F purpose="batch" \
  -F file="@batch_input.jsonl"
```

**Response (`200 OK`):**
```json
{
  "id": "file-1f8a7e3d-5b2c-491a-8f3e-112233445566",
  "object": "file",
  "bytes": 524,
  "created_at": 1789640000,
  "filename": "batch_input.jsonl",
  "purpose": "batch",
  "status": "processed"
}
```

### 3. Create the Batch (`POST /v1/batches`)

Submit the batch with your uploaded `file_id` and the `/v1/chat/completions` endpoint:

```bash
curl https://api.dos.ai/v1/batches \
  -H "Authorization: Bearer $DOS_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "input_file_id": "file-1f8a7e3d-5b2c-491a-8f3e-112233445566",
    "endpoint": "/v1/chat/completions",
    "completion_window": "24h"
  }'
```

**Response (`200 OK`):**
```json
{
  "id": "batch_98765432-abcd-ef01-2345-6789abcdef01",
  "object": "batch",
  "endpoint": "/v1/chat/completions",
  "input_file_id": "file-1f8a7e3d-5b2c-491a-8f3e-112233445566",
  "output_file_id": null,
  "completion_window": "24h",
  "status": "validating",
  "created_at": 1789640010,
  "expires_at": 1789726410,
  "in_progress_at": null,
  "completed_at": null,
  "request_counts": {
    "total": 3,
    "completed": 0,
    "failed": 0
  }
}
```

### 4. Check Batch Status (`GET /v1/batches/{batch_id}`)

Poll the batch status periodically (e.g. every 30 to 60 seconds):

```bash
curl https://api.dos.ai/v1/batches/batch_98765432-abcd-ef01-2345-6789abcdef01 \
  -H "Authorization: Bearer $DOS_API_KEY"
```

Lifecycle states:
- `validating`: Input file is being checked.
- `in_progress`: Requests are currently being processed.
- `completed`: All requests have been processed.
- `failed`: The batch encountered a fatal file or quota error.
- `cancelled`: The batch was cancelled by the user.

When status reaches `completed`, the response includes `output_file_id`:

```json
{
  "id": "batch_98765432-abcd-ef01-2345-6789abcdef01",
  "object": "batch",
  "endpoint": "/v1/chat/completions",
  "input_file_id": "file-1f8a7e3d-5b2c-491a-8f3e-112233445566",
  "output_file_id": "file-89abcdef-0123-4567-89ab-cdef01234567",
  "completion_window": "24h",
  "status": "completed",
  "created_at": 1789640010,
  "expires_at": 1789726410,
  "in_progress_at": 1789640030,
  "completed_at": 1789640350,
  "request_counts": {
    "total": 3,
    "completed": 3,
    "failed": 0
  }
}
```

### 5. Download Results (`GET /v1/files/{file_id}/content`)

Download the raw JSONL output containing completions for each `custom_id`:

```bash
curl https://api.dos.ai/v1/files/file-89abcdef-0123-4567-89ab-cdef01234567/content \
  -H "Authorization: Bearer $DOS_API_KEY" \
  -o batch_output.jsonl
```

Each line in `batch_output.jsonl` contains:
```json
{"id": "batch_req_batch_98765432-abcd-ef01-2345-6789abcdef01_req-1", "custom_id": "req-1", "response": {"status_code": 200, "request_id": "...", "body": {"choices": [{"message": {"role": "assistant", "content": "..."}}], "usage": {"prompt_tokens": 18, "completion_tokens": 85, "total_tokens": 103}}}, "error": null}
```

---

## Python SDK Example

You can use the official `openai` Python SDK by setting `base_url="https://api.dos.ai/v1"`:

```python
import time
from openai import OpenAI

# Initialize client pointing to DOS.AI Gateway
client = OpenAI(
    api_key="dos_sk_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx",
    base_url="https://api.dos.ai/v1",
)

# 1. Upload input file
batch_file = client.files.create(
    file=open("batch_input.jsonl", "rb"),
    purpose="batch"
)
print(f"Uploaded file ID: {batch_file.id}")

# 2. Create batch job
batch_job = client.batches.create(
    input_file_id=batch_file.id,
    endpoint="/v1/chat/completions",
    completion_window="24h"
)
print(f"Created batch ID: {batch_job.id}")

# 3. Poll until completion
while True:
    batch_status = client.batches.retrieve(batch_job.id)
    print(f"Status: {batch_status.status} (completed: {batch_status.request_counts.completed}/{batch_status.request_counts.total})")
    if batch_status.status in ("completed", "failed", "cancelled"):
        break
    time.sleep(30)

# 4. Retrieve results
if batch_status.status == "completed":
    content = client.files.content(batch_status.output_file_id)
    with open("results.jsonl", "wb") as f:
        f.write(content.read())
    print("Batch results saved to results.jsonl")
```

---

## Cancelling a Batch (`POST /v1/batches/{batch_id}/cancel`)

To cancel a running or validating batch:

```bash
curl -X POST https://api.dos.ai/v1/batches/batch_98765432-abcd-ef01-2345-6789abcdef01/cancel \
  -H "Authorization: Bearer $DOS_API_KEY"
```

Requests that have already completed before cancellation will remain recorded in the output file and billed; remaining requests are skipped immediately and will not consume tokens.
