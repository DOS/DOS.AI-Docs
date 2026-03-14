# InferenceSense-Like Alpha MVP

Updated: 2026-03-13
Status: Draft
Owner: DOS.AI

## Inspired By

This document is inspired by `InferenceSense`, the FriendliAI product described publicly as a way to monetize idle GPU capacity by running paid inference workloads on otherwise unused accelerator time.

At a high level, the public concept behind InferenceSense is:

- operators contribute idle GPU capacity
- an external control layer detects reclaimable windows
- paid inference demand is routed into that spare capacity
- workloads stop or get preempted when the operator needs the GPU back

This document does not attempt to specify the actual FriendliAI product. It defines a much smaller alpha that borrows the core idea:

- route short inference workloads into spare GPU capacity
- treat the service as best-effort
- give node owners a simple reclaim path

Important difference:

- `InferenceSense` appears aimed at operator-grade infrastructure and monetized demand
- this document defines a `friend-alpha` that can be built and tested by one person with trusted external nodes

## Why This Does Not Clone InferenceSense

This alpha intentionally does not try to replicate the full product shape of InferenceSense.

Reasons:

- the builder is one person, not an infra team
- the first goal is validating spare-capacity routing, not building a GPU marketplace
- there is no guaranteed external demand network yet
- operator onboarding is manual and trust-based
- the alpha should tolerate unstable nodes and best-effort behavior

What this means in practice:

- no Kubernetes dependency in v1
- no public operator onboarding
- no automated revenue sharing
- no promise of operator-grade reclaim latency
- no assumption that idle capacity is continuously available

## Goal

Build a first external alpha for a GPU spare-capacity inference service that can be tested by trusted friends.

This is not a public marketplace and not an enterprise platform. The goal is to validate:

- spare GPU detection
- routing inference requests to idle external nodes
- draining nodes when owners need their GPU back
- best-effort endpoint behavior for short requests

## Product Framing

This is a `friend-alpha / trusted operators alpha`, not an internal private cluster.

Constraints:

- one primary builder
- AI-assisted implementation
- no infra team
- no SLA
- no public onboarding
- no automated payouts

Success for v1 means a small number of external nodes can serve short inference requests through one shared endpoint without the system falling apart when operators reclaim their machines.

## Non-Goals

Do not build these in v1:

- public GPU marketplace
- automatic payouts or revenue sharing
- Kubernetes-based scheduling
- hard preemption in the middle of arbitrary long requests
- autoscaling
- multi-region failover
- tenant isolation beyond basic alpha safeguards
- arbitrary model pull and execution
- long-context serving

## Core User Types

### Operator

A trusted friend running a GPU node.

Capabilities:

- install node agent
- register node with control server
- enable or disable `spare mode`
- view basic node health
- reclaim GPU when local work returns

### Tester

A trusted alpha user calling a shared inference endpoint.

Expectations:

- OpenAI-compatible API
- best-effort behavior
- short requests only
- occasional failures are acceptable during alpha

### Control Server Admin

The builder operating the alpha.

Responsibilities:

- whitelist operators
- issue API keys
- observe node health
- inspect request failures
- tune admission thresholds

## MVP Architecture

The simplest deployable architecture has four pieces:

1. `Node Agent`
2. `Control Plane API`
3. `Gateway / Router`
4. `Inference Node (vLLM on operator machine)`

### High-Level Flow

```text
Operator machine
  Node Agent -> Heartbeat + capacity -> Control Plane
  vLLM server <- Gateway forwards request

Tester
  -> Gateway API
  -> Router selects available node
  -> Request forwarded to operator node

Operator reclaims GPU
  -> Agent sets node to draining
  -> Gateway stops routing new requests
  -> In-flight request gets short grace window
```

## Component Design

### 1. Node Agent

Runs on the operator machine and acts as the local executor.

Responsibilities:

- register node
- send heartbeat every 2-5 seconds
- report GPU metrics
- report current status
- manage `spare mode`
- switch node into `draining`
- optionally start/stop local vLLM container

Minimum metrics:

- GPU name
- total VRAM
- used VRAM
- GPU utilization
- local node status: `available`, `draining`, `offline`
- model served
- recent heartbeat timestamp

Recommended implementation:

- Python
- `pynvml` for GPU stats
- Docker SDK or shell commands for local container management

### 2. Control Plane API

Central source of truth for nodes and request metadata.

Responsibilities:

- node registration
- heartbeat ingestion
- node status persistence
- API key validation
- request logging
- usage tracking

Recommended implementation:

- FastAPI
- Postgres or SQLite for alpha
- Redis optional for ephemeral routing state

### 3. Gateway / Router

Single endpoint exposed to testers.

Responsibilities:

- authenticate tester requests
- validate request shape
- enforce alpha limits
- select target node
- forward request to node vLLM endpoint
- return model response
- log request status and latency

Recommended behavior:

- OpenAI-compatible `/v1/chat/completions`
- model whitelist
- hard cap on `max_tokens`
- request timeout
- no streaming in earliest cut unless needed immediately

### 4. Inference Node

vLLM server running on the operator machine.

Alpha simplifications:

- one node serves one model
- no dynamic model switching
- no multi-model hot loading
- only approved models

## Node State Machine

Keep the first version explicit and simple.

States:

- `offline`
- `available`
- `busy`
- `draining`
- `error`

Transitions:

- `offline -> available`: heartbeat received and spare mode enabled
- `available -> busy`: request assigned
- `busy -> available`: request completed
- `available -> draining`: operator reclaim or threshold crossed
- `busy -> draining`: operator reclaim while request in-flight
- `draining -> available`: operator re-enables spare mode and node healthy
- `any -> error`: agent or vLLM health failure
- `any -> offline`: heartbeat timeout

Routing rules:

- only route to `available`
- never route to `draining`
- mark stale nodes `offline` after short heartbeat timeout

## Admission Control

Admission control is mandatory in v1. This is what keeps the system usable.

Only accept requests that satisfy all of:

- allowed model
- prompt size below configured limit
- `max_tokens` below configured limit
- total estimated request time below configured limit
- at least one healthy node with enough spare capacity

Recommended alpha limits:

- prompt length: short to medium only
- `max_tokens <= 512`
- request timeout: `<= 20s`
- no very long context requests

## Routing Heuristic

Use a simple score instead of a complex scheduler.

Suggested routing score:

```text
score =
  model_match
  + heartbeat_freshness_bonus
  + low_gpu_util_bonus
  + low_vram_used_bonus
  - draining_penalty
  - recent_failure_penalty
```

Hard filters:

- heartbeat too old
- node not `available`
- GPU utilization above threshold
- free VRAM below threshold
- model mismatch

## Spare Capacity Heuristic

Do not overfit this in v1. The goal is "good enough to avoid obvious bad routing."

Example:

```text
spare_score = 100
  - (gpu_util_percent * 0.7)
  - (vram_used_ratio * 30)
  - foreground_owner_activity_penalty
```

Initial thresholds can be crude:

- GPU util must stay below `60-70%`
- sufficient free VRAM for the configured model
- node must not be in reclaim mode

## Preemption and Draining

Do not implement true hard preemption in week 1.

Implement `draining`:

- node stops accepting new requests immediately
- in-flight request gets a short grace window
- if request completes inside grace window, node returns idle
- if not, request is terminated and logged as interrupted

Recommended grace window:

- `10-20s`

This is enough to validate the operator reclaim concept without taking on complex mid-generation recovery.

## API Surface

### Control Plane

`POST /nodes/register`

- register a node

`POST /nodes/heartbeat`

- update capacity and status

`POST /nodes/{id}/mode`

- switch `spare mode` on or off

`GET /nodes`

- list nodes and health

`GET /requests`

- list recent request records

### Gateway

`POST /v1/chat/completions`

- OpenAI-compatible alpha endpoint

`GET /health`

- gateway health

`GET /models`

- list available alpha models

## Data Model

### `nodes`

- `id`
- `name`
- `owner_name`
- `status`
- `public_url`
- `gpu_name`
- `vram_total_mb`
- `current_model`
- `last_heartbeat_at`

### `node_capacity`

- `node_id`
- `gpu_util_percent`
- `vram_used_mb`
- `vram_free_mb`
- `spare_score`
- `is_accepting_jobs`
- `updated_at`

### `api_keys`

- `id`
- `label`
- `key_hash`
- `rate_limit_per_min`
- `enabled`

### `requests`

- `id`
- `api_key_id`
- `node_id`
- `model`
- `status`
- `prompt_tokens_est`
- `max_tokens`
- `latency_ms`
- `error_code`
- `created_at`

## Security for Alpha

Keep this basic but not careless.

Required:

- per-node auth token
- per-tester API key
- allowlist operators manually
- allowlist models manually
- HTTPS on public endpoints

Explicitly postponed:

- mTLS
- operator self-serve onboarding
- strong multi-tenant isolation
- arbitrary untrusted code execution

## Observability

Minimum required telemetry:

- node online/offline count
- node status changes
- request count
- success/failure count
- average latency
- interrupted request count

Initial implementation can be:

- structured logs
- basic request table
- simple admin JSON endpoints

Do not block v1 on a full dashboard.

## Recommended Tech Stack

### Control Plane and Gateway

- Python
- FastAPI
- SQLAlchemy
- Postgres or SQLite
- Redis optional

### Node Agent

- Python
- `pynvml`
- Docker SDK or shell execution

### Inference

- vLLM
- one approved model per node

## Suggested Repo Layout

```text
/apps/control-plane
/apps/gateway
/apps/node-agent
/packages/shared
/docs
```

If speed matters more than repo purity, `control-plane` and `gateway` can start as one service and split later.

## 7-Day Build Plan

### Day 1

- scaffold control-plane service
- define DB schema
- implement node registration
- implement heartbeat endpoint

### Day 2

- build node agent
- collect GPU stats
- send heartbeat
- implement `spare mode`

### Day 3

- build OpenAI-compatible gateway endpoint
- add API key auth
- forward request to a selected node

### Day 4

- add routing logic
- add admission control
- add request timeout and logging

### Day 5

- add draining flow
- add reclaim path
- add node health checks

### Day 6

- add admin endpoints
- inspect nodes
- inspect recent requests
- inspect failure reasons

### Day 7

- deploy alpha
- onboard 2-3 trusted nodes
- run live tests
- tune thresholds and request caps

## Success Criteria

The alpha is good enough if all of these are true:

- 2-3 external nodes can stay online reliably
- at least one shared endpoint works end-to-end
- short requests complete successfully most of the time
- node reclaim does not break the entire system
- failures are observable and diagnosable

## Risks

### Technical Risks

- node public connectivity problems
- vLLM cold start latency
- request interruption during reclaim
- poor spare-capacity detection
- unstable home-network operators

### Product Risks

- testers expect stable SLA from a best-effort service
- operators forget to disable spare mode
- alpha users send prompts that exceed safe caps

## Hard Cuts for v1

If the build slips, cut these first:

- streaming responses
- admin UI
- Redis
- multi-model support
- fancy routing heuristics

Do not cut:

- heartbeat
- node auth
- draining
- request logging
- admission limits

## Immediate Next Spec

After this document, the next useful artifact is:

1. API contract doc
2. node-agent state machine doc
3. request lifecycle and failure-mode test matrix

These should be written before broad AI-assisted implementation starts, so the generated code does not drift into a fake platform architecture.
