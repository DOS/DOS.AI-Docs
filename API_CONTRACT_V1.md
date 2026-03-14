# API Contract V1

Updated: 2026-03-13
Status: Draft
Owner: DOS.AI
Related: `INFERENCESENSE_LIKE_ALPHA_MVP.md`

## Purpose

This document defines the API contract for the first `InferenceSense-like` alpha.

Its job is to freeze:

- endpoint surface
- request and response shapes
- auth expectations
- enum values
- error behavior

This is a v1 alpha contract. It is intentionally narrow.

## Scope

This contract covers three communication paths:

1. `Node Agent -> Control Plane`
2. `Tester -> Gateway`
3. `Admin -> Control Plane`

It does not define:

- billing APIs
- payout APIs
- public operator onboarding
- marketplace APIs
- long-running job APIs

## General Rules

### Transport

- JSON over HTTPS
- UTF-8
- request body and response body are JSON unless explicitly stated otherwise

### Time

- all timestamps are ISO 8601 UTC strings
- example: `2026-03-13T08:15:30Z`

### IDs

- `node_id`, `request_id`, and `api_key_id` are opaque strings
- UUIDv7 is recommended but not required by the contract

### Error Envelope

Unless otherwise specified, errors use this shape:

```json
{
  "error": {
    "code": "NO_AVAILABLE_NODE",
    "message": "No available node can serve this request.",
    "retryable": true
  }
}
```

Fields:

- `code`: stable machine-readable code
- `message`: human-readable summary
- `retryable`: whether the client may retry later

## Enums

### Node Status

Allowed values:

- `offline`
- `available`
- `busy`
- `draining`
- `error`

### Node Mode

Allowed values:

- `spare_on`
- `spare_off`

### Request Status

Allowed values:

- `queued`
- `assigned`
- `running`
- `completed`
- `failed`
- `interrupted`
- `rejected`

### Request Error Codes

Allowed values in v1:

- `NO_AVAILABLE_NODE`
- `NODE_STALE`
- `NODE_DRAINING`
- `NODE_UNHEALTHY`
- `MODEL_NOT_ALLOWED`
- `PROMPT_TOO_LARGE`
- `MAX_TOKENS_TOO_LARGE`
- `REQUEST_TIMEOUT`
- `FORWARDED_REQUEST_FAILED`
- `REQUEST_INTERRUPTED`
- `INVALID_API_KEY`
- `INVALID_NODE_TOKEN`
- `RATE_LIMITED`
- `BAD_REQUEST`

## Authentication

### Node Agent -> Control Plane

Use header:

```text
Authorization: Bearer <node_token>
```

The `node_token` is issued manually by the admin during alpha onboarding.

### Tester -> Gateway

Use header:

```text
Authorization: Bearer <api_key>
```

### Admin APIs

For alpha, admin auth may be:

- a separate static admin token
- or an allowlisted reverse proxy

The exact auth mechanism can stay implementation-specific as long as admin APIs are not public.

## Control Plane APIs

### POST `/nodes/register`

Registers a node or refreshes node metadata for a trusted operator.

Auth:

- required
- `Authorization: Bearer <node_token>`

Request:

```json
{
  "node_name": "joy-rtx6000-01",
  "owner_name": "JOY",
  "public_base_url": "https://node-01.example.com",
  "gpu_name": "RTX Pro 6000 Blackwell",
  "vram_total_mb": 98304,
  "current_model": "Qwen/Qwen3.5-35B-A3B-GPTQ-Int4",
  "agent_version": "0.1.0"
}
```

Rules:

- `public_base_url` must be reachable by the gateway
- `current_model` must be in the control-plane allowlist
- one node serves one model in v1

Success response:

```json
{
  "node_id": "node_01HXYZ...",
  "status": "offline",
  "accepted_model": "Qwen/Qwen3.5-35B-A3B-GPTQ-Int4",
  "heartbeat_interval_sec": 5
}
```

Errors:

- `401 INVALID_NODE_TOKEN`
- `400 BAD_REQUEST`
- `400 MODEL_NOT_ALLOWED`

### POST `/nodes/heartbeat`

Updates live node capacity and status.

Auth:

- required
- `Authorization: Bearer <node_token>`

Request:

```json
{
  "node_id": "node_01HXYZ...",
  "status": "available",
  "mode": "spare_on",
  "gpu_util_percent": 22.4,
  "vram_used_mb": 24576,
  "vram_free_mb": 73728,
  "spare_score": 78.2,
  "is_accepting_jobs": true,
  "active_request_count": 0,
  "last_local_error": null,
  "observed_at": "2026-03-13T08:15:30Z"
}
```

Rules:

- `status` must be one of the allowed node states
- `is_accepting_jobs` must be `false` when `status = draining`
- heartbeat interval should be `2-5s`

Success response:

```json
{
  "ok": true,
  "server_time": "2026-03-13T08:15:30Z",
  "effective_status": "available",
  "should_drain": false
}
```

Notes:

- `effective_status` lets the server override bad local assumptions
- `should_drain = true` can be used later to signal server-side admission controls

Errors:

- `401 INVALID_NODE_TOKEN`
- `404 BAD_REQUEST`

### POST `/nodes/{node_id}/mode`

Switches node spare mode.

Auth:

- required
- `Authorization: Bearer <node_token>`

Request:

```json
{
  "mode": "spare_off",
  "reason": "owner_reclaim"
}
```

Rules:

- `spare_off` implies `draining` or `offline`
- `spare_on` does not guarantee `available`; the node must still pass health checks

Success response:

```json
{
  "node_id": "node_01HXYZ...",
  "mode": "spare_off",
  "status": "draining"
}
```

Errors:

- `401 INVALID_NODE_TOKEN`
- `404 BAD_REQUEST`

### GET `/nodes`

Lists nodes and current health.

Auth:

- admin only

Success response:

```json
{
  "nodes": [
    {
      "node_id": "node_01HXYZ...",
      "node_name": "joy-rtx6000-01",
      "owner_name": "JOY",
      "status": "available",
      "mode": "spare_on",
      "current_model": "Qwen/Qwen3.5-35B-A3B-GPTQ-Int4",
      "gpu_util_percent": 22.4,
      "vram_free_mb": 73728,
      "spare_score": 78.2,
      "active_request_count": 0,
      "last_heartbeat_at": "2026-03-13T08:15:30Z"
    }
  ]
}
```

### GET `/requests`

Lists recent request records.

Auth:

- admin only

Query params:

- `limit` optional
- `status` optional
- `node_id` optional

Success response:

```json
{
  "requests": [
    {
      "request_id": "req_01HXYZ...",
      "node_id": "node_01HXYZ...",
      "model": "Qwen/Qwen3.5-35B-A3B-GPTQ-Int4",
      "status": "completed",
      "prompt_tokens_est": 384,
      "max_tokens": 256,
      "latency_ms": 4820,
      "error_code": null,
      "created_at": "2026-03-13T08:16:10Z"
    }
  ]
}
```

## Gateway APIs

### GET `/health`

Public health for gateway availability.

Success response:

```json
{
  "ok": true,
  "service": "gateway",
  "time": "2026-03-13T08:15:30Z"
}
```

### GET `/models`

Returns available models currently routable by the alpha.

Auth:

- optional in v1
- may be protected if desired

Success response:

```json
{
  "data": [
    {
      "id": "Qwen/Qwen3.5-35B-A3B-GPTQ-Int4",
      "object": "model",
      "owned_by": "dos-ai-alpha"
    }
  ]
}
```

### POST `/v1/chat/completions`

Primary tester-facing endpoint.

This is OpenAI-compatible only to the extent needed by the alpha. Full parity is out of scope.

Auth:

- required
- `Authorization: Bearer <api_key>`

Request:

```json
{
  "model": "Qwen/Qwen3.5-35B-A3B-GPTQ-Int4",
  "messages": [
    {
      "role": "system",
      "content": "You are a concise assistant."
    },
    {
      "role": "user",
      "content": "Summarize the following URL risk signals."
    }
  ],
  "temperature": 0.2,
  "max_tokens": 256,
  "stream": false
}
```

Alpha rules:

- `model` must be allowlisted
- `messages` must not exceed configured prompt size cap
- `max_tokens` must not exceed configured cap
- `stream` may be accepted only if implemented; otherwise reject

Success response:

```json
{
  "id": "chatcmpl_01HXYZ...",
  "object": "chat.completion",
  "created": 1773389730,
  "model": "Qwen/Qwen3.5-35B-A3B-GPTQ-Int4",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "The page shows several phishing indicators."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 112,
    "completion_tokens": 58,
    "total_tokens": 170
  }
}
```

Error responses:

`401 INVALID_API_KEY`

```json
{
  "error": {
    "code": "INVALID_API_KEY",
    "message": "API key is invalid.",
    "retryable": false
  }
}
```

`400 MODEL_NOT_ALLOWED`

```json
{
  "error": {
    "code": "MODEL_NOT_ALLOWED",
    "message": "Requested model is not available in this alpha.",
    "retryable": false
  }
}
```

`400 PROMPT_TOO_LARGE`

```json
{
  "error": {
    "code": "PROMPT_TOO_LARGE",
    "message": "Prompt exceeds the alpha size limit.",
    "retryable": false
  }
}
```

`400 MAX_TOKENS_TOO_LARGE`

```json
{
  "error": {
    "code": "MAX_TOKENS_TOO_LARGE",
    "message": "max_tokens exceeds the alpha limit.",
    "retryable": false
  }
}
```

`429 RATE_LIMITED`

```json
{
  "error": {
    "code": "RATE_LIMITED",
    "message": "Rate limit exceeded.",
    "retryable": true
  }
}
```

`503 NO_AVAILABLE_NODE`

```json
{
  "error": {
    "code": "NO_AVAILABLE_NODE",
    "message": "No available node can serve this request right now.",
    "retryable": true
  }
}
```

`504 REQUEST_TIMEOUT`

```json
{
  "error": {
    "code": "REQUEST_TIMEOUT",
    "message": "The request exceeded the alpha timeout.",
    "retryable": true
  }
}
```

`502 FORWARDED_REQUEST_FAILED`

```json
{
  "error": {
    "code": "FORWARDED_REQUEST_FAILED",
    "message": "The selected node failed while processing the request.",
    "retryable": true
  }
}
```

`503 REQUEST_INTERRUPTED`

```json
{
  "error": {
    "code": "REQUEST_INTERRUPTED",
    "message": "The request was interrupted because the node was reclaimed.",
    "retryable": true
  }
}
```

## Routing and Assignment Behavior

This contract keeps routing internal, but the following behavior is part of v1 expectations:

- only `available` nodes can receive new requests
- `draining` nodes must not receive new requests
- stale heartbeats must cause the node to be excluded
- the gateway may select a node using simple heuristics
- a failed forwarded request may be retried once only if retry logic is implemented

For v1, the gateway is allowed to:

- reject instead of queueing
- fail fast when no safe node exists

The gateway is not required to:

- hold long queues
- guarantee fairness
- preserve session affinity

## Request Lifecycle

Canonical flow:

1. gateway validates API key
2. gateway validates payload limits
3. gateway selects node
4. gateway creates request record with `assigned`
5. gateway forwards request to node
6. request transitions to `running`
7. request ends as `completed`, `failed`, or `interrupted`

## Heartbeat Staleness Rules

Recommended v1 defaults:

- expected heartbeat interval: `5s`
- stale after: `10s`
- offline after: `15s`

Behavior:

- stale nodes are excluded from routing
- offline nodes are shown as unavailable in admin views

## Streaming

Streaming is optional in v1.

If not implemented:

- reject `stream=true` with `400 BAD_REQUEST`

If implemented later:

- preserve the same auth, model validation, and error semantics as non-streaming requests

## Compatibility Notes

This API is only partially OpenAI-compatible.

Guaranteed in v1:

- `/v1/chat/completions`
- `model`
- `messages`
- `temperature`
- `max_tokens`
- `stream` field acceptance or explicit rejection

Not guaranteed in v1:

- tools
- function calling
- response_format
- logprobs
- seed
- strict response parity with OpenAI

## Implementation Notes

To reduce drift across services:

- define shared Pydantic models for request and response schemas
- define shared enums for node and request statuses
- define shared error code constants

Do not allow each service to invent its own field names.

## Immediate Follow-Up Docs

After this contract, the next useful documents are:

1. `NODE_AGENT_STATE_MACHINE_V1.md`
2. `FAILURE_TEST_MATRIX_V1.md`

These should define reclaim behavior, heartbeat failure handling, and interrupt cases before implementation expands.
