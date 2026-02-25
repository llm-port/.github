# llm.port Call Sequence Graphs

This document captures end-to-end call sequences across frontend, backend, gateway, and shared stack containers.

## 1) End-to-End User + Control Plane + Data Plane

```mermaid
sequenceDiagram
    autonumber
    participant U as Operator / App Client
    participant FE as llm_port_frontend (llm_port_frontend)
    participant BE as llm_port_backend (llm_port_backend)
    participant GW as llm_port_api (Gateway)
    participant RAG as llm_port_rag (Internal API)
    participant PG as Postgres
    participant R as Redis
    participant RMQ as RabbitMQ
    participant MIO as MinIO
    participant LP as LLM Provider Runtime/API
    participant LF as Langfuse (web/worker)
    participant LK as Loki
    participant GF as Grafana
    participant DE as Docker Engine / Compose

    rect rgb(245,245,255)
    U->>FE: Open Admin Console
    FE->>BE: Authenticated /api/* admin calls (RBAC)
    BE-->>FE: Admin views + permissions + data
    end

    rect rgb(245,255,245)
    U->>GW: OpenAI API call (/v1/models|chat|embeddings)
    GW->>R: Rate limit + lease checks
    GW->>PG: Tenant policy + model/pool metadata + audit log
    GW->>LP: Upstream OpenAI-compatible request
    LP-->>GW: Completion / stream / embeddings response
    GW->>LF: Trace + generation + usage + error status
    GW-->>U: OpenAI-compatible response/error envelope
    end

    rect rgb(245,245,235)
    FE->>BE: RAG admin/search calls (/api/admin/rag/*)
    BE->>RAG: Proxy internal request (/api/internal/* + service token)
    RAG->>PG: Read/write RAG metadata + vectors
    RAG->>MIO: Read/write raw snapshots and uploads
    RAG->>RMQ: Enqueue ingest/publish tasks
    RMQ-->>RAG: Task delivery to worker/scheduler
    RAG-->>BE: Internal response
    BE-->>FE: RAG response payload
    end

    rect rgb(255,245,245)
    FE->>BE: Logs/metrics/traces UI calls
    BE->>PG: Read graph trace rows (llm_gateway_request_log)
    BE->>LK: Query logs
    BE-->>FE: Graph data + logs
    FE->>GF: Open embedded dashboards
    GF->>LK: Query log datasource
    end

    rect rgb(245,250,245)
    FE->>BE: Container/service control actions
    BE->>DE: Start/stop/restart/recreate service
    DE-->>BE: Execution result
    BE-->>FE: Action status + audit
    end
```

## 2) Gateway `/v1/chat/completions` (Non-Stream)

```mermaid
sequenceDiagram
    autonumber
    participant C as API Client
    participant GW as llm_port_api
    participant AUTH as JWT Auth
    participant DAO as GatewayDAO (Postgres: llm_api)
    participant RL as Redis Rate Limit
    participant LEASE as Redis Lease
    participant UP as Upstream Provider
    participant OBS as Langfuse

    C->>GW: POST /v1/chat/completions (stream=false)
    GW->>AUTH: Verify Bearer token
    AUTH-->>GW: sub + tenant_id claims
    GW->>DAO: Load tenant policy + alias pool
    GW->>RL: Check RPM/TPM window counters
    RL-->>GW: allow/deny
    alt Allowed
      GW->>LEASE: Acquire provider lease
      LEASE-->>GW: lease granted
      GW->>OBS: Start trace/generation
      GW->>UP: Proxy OpenAI-compatible request
      alt Pre-first-token failure
        GW->>UP: Retry once (MVP)
      end
      UP-->>GW: JSON completion
      GW->>OBS: Finalize success/failure + usage/latency
      GW->>DAO: Insert audit log row (trace_id, usage, status)
      GW->>LEASE: Release lease (finally)
      GW-->>C: OpenAI-compatible JSON
    else Rejected
      GW-->>C: OpenAI error envelope (401/403/429/etc.)
    end
```

## 3) Gateway `/v1/chat/completions` (Stream SSE)

```mermaid
sequenceDiagram
    autonumber
    participant C as API Client
    participant GW as llm_port_api
    participant DAO as GatewayDAO
    participant RL as Redis Rate Limit
    participant LEASE as Redis Lease
    participant UP as Upstream Provider
    participant OBS as Langfuse

    C->>GW: POST /v1/chat/completions (stream=true)
    GW->>DAO: Auth + policy + routing data
    GW->>RL: Check RPM/TPM
    GW->>LEASE: Acquire lease
    GW->>OBS: Start trace
    GW->>UP: Open upstream SSE
    loop Each chunk
      UP-->>GW: data: {chat.completion.chunk}
      GW-->>C: data: {chat.completion.chunk}
    end
    UP-->>GW: data: [DONE]
    GW-->>C: data: [DONE]
    GW->>OBS: Finalize stream (TTFT/usage/latency)
    GW->>DAO: Insert audit log (status/ttft/tokens)
    GW->>LEASE: Release lease (finally)
```

## 4) Gateway `/v1/embeddings`

```mermaid
sequenceDiagram
    autonumber
    participant C as API Client
    participant GW as llm_port_api
    participant DAO as GatewayDAO
    participant R as Redis
    participant UP as Upstream Provider
    participant OBS as Langfuse

    C->>GW: POST /v1/embeddings
    GW->>DAO: Auth + tenant policy + alias resolution
    GW->>R: RPM/TPM + concurrency lease
    GW->>OBS: Start trace
    GW->>UP: Proxy embeddings request
    UP-->>GW: Embeddings response
    GW->>OBS: Finalize (metadata + usage, no vector payload)
    GW->>DAO: Audit log write
    GW-->>C: OpenAI-compatible embeddings JSON
```

## 5) LLM Graph Page (Topology + Live Traces)

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin User
    participant FE as llm_port_frontend (/admin/llm/graph)
    participant BE as llm_port_backend
    participant SVC as graph_service
    participant PG as Postgres (llm_api + backend DB)

    U->>FE: Open Agent Trace page
    FE->>BE: GET /api/llm/graph/topology
    BE->>SVC: get_topology()
    SVC->>PG: Read provider/runtime/model registry
    PG-->>SVC: Topology rows
    SVC-->>BE: Normalized nodes/edges
    BE-->>FE: TopologyResponse

    FE->>BE: GET /api/llm/graph/traces?limit=100
    BE->>SVC: list_recent_traces(limit)
    SVC->>PG: Read llm_gateway_request_log
    PG-->>SVC: Trace events
    SVC-->>BE: TraceSnapshotResponse
    BE-->>FE: Snapshot

    FE->>BE: GET /api/llm/graph/traces/stream (SSE)
    loop New trace event
      BE->>SVC: stream_traces(cursor)
      SVC->>PG: Poll/read newer events
      PG-->>SVC: Delta events
      SVC-->>BE: trace event
      BE-->>FE: event: trace\nid: <event_id>\ndata: {...}
    end
    BE-->>FE: event: ping (keepalive)
```

## 6) Container Management (Admin Ops)

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin User
    participant FE as llm_port_frontend
    participant BE as llm_port_backend
    participant RBAC as Auth + RBAC + RootMode
    participant OPS as Container Service
    participant DE as Docker Engine

    U->>FE: Start/Stop/Restart container action
    FE->>BE: POST /api/admin/containers/{id}/{action}
    BE->>RBAC: Validate token + permission + root mode (if needed)
    RBAC-->>BE: allow/deny
    alt Allowed
      BE->>OPS: Execute action
      OPS->>DE: Docker API call
      DE-->>OPS: Result / error
      OPS-->>BE: Action status
      BE-->>FE: 200 + operation result
    else Denied
      BE-->>FE: 401/403 error
    end
```

## 7) System Settings (Immediate Apply)

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin User
    participant FE as llm_port_frontend (Settings)
    participant BE as llm_port_backend
    participant SET as system_settings service
    participant ENC as Secret Crypto
    participant DB as Backend DB
    participant APPLY as Apply Orchestrator
    participant EXEC as LocalExecutor
    participant DE as Docker Compose/Engine

    U->>FE: Update setting key/value
    FE->>BE: PUT /api/admin/system/settings/values/{key}
    BE->>SET: Validate schema + impact scope
    alt Secret field
      SET->>ENC: Encrypt secret
      ENC-->>SET: ciphertext + metadata
      SET->>DB: Save system_setting_secret
    else Non-secret field
      SET->>DB: Save system_setting_value
    end

    alt apply_scope == live_reload
      SET-->>BE: Applied in-process
      BE-->>FE: Updated (no job)
    else service_restart or stack_recreate
      SET->>APPLY: Create immediate apply job
      APPLY->>DB: Insert system_apply_job + events
      APPLY->>EXEC: apply(change_set)
      EXEC->>DE: restart/recreate services
      DE-->>EXEC: per-service results
      EXEC-->>APPLY: success/failure
      alt Failure
        APPLY->>EXEC: rollback(previous snapshot)
      end
      APPLY->>DB: Persist final status/events
      BE-->>FE: Updated + job_id
      FE->>BE: GET /api/admin/system/apply/{job_id} (poll)
      BE-->>FE: Job status/events
    end
```

## 8) System Initialization Wizard

```mermaid
sequenceDiagram
    autonumber
    participant U as Operator
    participant FE as llm_port_frontend (Wizard)
    participant BE as llm_port_backend
    participant WIZ as Wizard Service
    participant SET as Settings Service
    participant APPLY as Apply Orchestrator

    U->>FE: Open System Init Wizard
    FE->>BE: GET /api/admin/system/wizard/steps
    BE->>WIZ: Load step schema
    WIZ-->>BE: Step metadata
    BE-->>FE: Steps (1..6)

    loop Per step submission
      FE->>BE: POST /api/admin/system/wizard/apply
      BE->>WIZ: Validate step payload
      WIZ->>SET: Write settings via shared path
      SET->>APPLY: Trigger immediate apply when required
      APPLY-->>SET: job refs + status
      SET-->>WIZ: per-setting result
      WIZ-->>BE: step result
      BE-->>FE: step status + errors + job ids
    end
```

## 9) Multi-Host Agent Contract (Feature-Flagged Path)

```mermaid
sequenceDiagram
    autonumber
    participant AG as Infra Agent
    participant BE as llm_port_backend
    participant REG as Agent Registry
    participant APPLY as Apply Orchestrator

    AG->>BE: POST /api/admin/system/agents/register
    BE->>REG: Validate token + save capabilities
    REG-->>BE: agent_id
    BE-->>AG: Registered

    loop Heartbeat interval
      AG->>BE: POST /api/admin/system/agents/heartbeat
      BE->>REG: Update status/last_seen
      REG-->>BE: ok
      BE-->>AG: ack
    end

    BE->>AG: POST /api/admin/system/agents/{id}/apply
    AG-->>BE: Accepted + remote_job_id
    loop Job sync
      BE->>AG: GET /api/admin/system/agents/{id}/jobs/{job_id}
      AG-->>BE: running/success/failure + events
    end
```

## 10) Logging + Observability Data Path

```mermaid
sequenceDiagram
    autonumber
    participant APP as Backend/Frontend/Gateway Containers
    participant AL as Alloy
    participant LK as Loki
    participant GF as Grafana
    participant GW as llm_port_api
    participant LF as Langfuse

    APP-->>AL: Container stdout/stderr logs
    AL-->>LK: Push structured log streams
    GF->>LK: Query logs/dashboard panels
    LK-->>GF: Log lines/time-series

    GW->>LF: LLM trace/generation events
    LF-->>GW: ingest ack
```

## 11) RAG Search (Frontend to Internal RAG)

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin User
    participant FE as llm_port_frontend (/admin/rag/search)
    participant BE as llm_port_backend
    participant RBAC as JWT + RBAC
    participant RC as RagServiceClient
    participant RAG as llm_port_rag
    participant DB as Postgres (RAG metadata + pgvector)

    U->>FE: Submit query + tenant/workspace/principals/filters
    FE->>BE: POST /api/admin/rag/knowledge/search
    BE->>RBAC: Check rag.search:read
    RBAC-->>BE: allow/deny
    alt Allowed
      BE->>RC: search_knowledge(payload)
      RC->>RAG: POST /api/internal/knowledge/search (service token)
      RAG->>DB: ACL + scope prefilter + vector/keyword/hybrid query
      DB-->>RAG: Ranked chunks
      RAG-->>RC: Search response
      RC-->>BE: Search response
      BE-->>FE: 200 results + optional debug
    else Denied
      BE-->>FE: 401/403
    end
```

## 12) RAG Upload + Draft + Publish (Immediate)

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin User
    participant FE as llm_port_frontend (/admin/rag/explorer)
    participant BE as llm_port_backend
    participant RAG as llm_port_rag API
    participant MIO as MinIO
    participant RMQ as RabbitMQ
    participant W as RAG Taskiq Worker
    participant DB as Postgres (RAG metadata + pgvector)

    U->>FE: Upload files into container
    FE->>BE: POST /api/admin/rag/uploads/presign
    BE->>RAG: POST /api/internal/admin/uploads/presign
    RAG-->>BE: object_key + presigned URL
    BE-->>FE: presigned URL
    FE->>MIO: PUT file bytes (direct browser upload)
    FE->>BE: POST /api/admin/rag/uploads/complete
    BE->>RAG: POST /api/internal/admin/uploads/complete
    RAG->>DB: Add draft operation (upload)
    RAG-->>BE: draft_id + operation_id
    BE-->>FE: Draft update

    U->>FE: Publish now
    FE->>BE: POST /api/admin/rag/drafts/{id}/publish
    BE->>RAG: POST /api/internal/admin/drafts/{id}/publish
    RAG->>DB: Create publish request + ingest job
    RAG->>RMQ: Enqueue rag.process_publish
    RMQ-->>W: Deliver task
    W->>MIO: Read uploaded object
    W->>DB: changed-file check (sha256), extract/chunk/embed/index
    W->>DB: Update draft/publish/job status + events
    RAG-->>BE: publish_id + job_id
    BE-->>FE: queued/running response
```

## 13) RAG Scheduled Publish

```mermaid
sequenceDiagram
    autonumber
    participant FE as llm_port_frontend
    participant BE as llm_port_backend
    participant RAG as llm_port_rag API
    participant S as RAG Scheduler
    participant RMQ as RabbitMQ
    participant W as RAG Taskiq Worker
    participant DB as Postgres (RAG metadata + pgvector)

    FE->>BE: POST /api/admin/rag/drafts/{id}/publish (scheduled_for)
    BE->>RAG: POST /api/internal/admin/drafts/{id}/publish
    RAG->>DB: Create publish request status=scheduled
    RAG-->>BE: publish_id status=scheduled
    BE-->>FE: Accepted scheduled publish

    loop scheduler tick
      S->>DB: Query due publishes (scheduled_for <= now)
      DB-->>S: due publish list
      S->>DB: Create ingest job + mark queued
      S->>RMQ: Enqueue rag.process_publish
    end

    RMQ-->>W: Deliver publish task
    W->>DB: Process operations and index changes
    W->>DB: Mark publish/job final status
```
