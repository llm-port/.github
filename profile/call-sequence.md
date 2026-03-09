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

## 14) Console Chat Completions (Frontend → Backend Proxy → Gateway)

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin User
    participant FE as llm_port_frontend (/chat)
    participant BE as llm_port_backend
    participant GW as llm_port_api
    participant PII as llm_port_pii
    participant LP as LLM Provider

    U->>FE: Send chat message (project + session)
    FE->>BE: POST /api/chat/completions (cookie auth)
    BE->>BE: Cookie → JWT conversion
    BE->>GW: POST /v1/chat/completions (Bearer JWT, session_id, project_id)
    GW->>GW: Inject memory facts + session context into system prompt
    GW->>GW: Resolve model alias → provider
    alt Cloud provider + PII policy active
      GW->>PII: POST /api/pii/sanitize (redact/tokenize)
      PII-->>GW: Sanitized payload
    end
    GW->>LP: Upstream chat request (stream=true)
    loop SSE chunks
      LP-->>GW: data: {chunk}
      alt PII tokenize mode
        GW->>PII: POST /api/pii/detokenize (response chunk)
        PII-->>GW: Restored text
      end
      GW-->>BE: data: {chunk}
      BE-->>FE: data: {chunk}
    end
    LP-->>GW: data: [DONE]
    GW->>GW: Persist message to session + audit log
    GW-->>BE: data: [DONE]
    BE-->>FE: data: [DONE]
    FE->>FE: Render streamed response
```

## 15) Session & Memory Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin / App Client
    participant GW as llm_port_api
    participant DB as Postgres (llm_api)

    rect rgb(245,245,255)
    Note over U,DB: Project + Session CRUD
    U->>GW: POST /v1/sessions/projects (create project)
    GW->>DB: Insert project row
    GW-->>U: project_id

    U->>GW: POST /v1/sessions (create session under project)
    GW->>DB: Insert session row
    GW-->>U: session_id

    U->>GW: POST /v1/chat/completions (session_id in body)
    GW->>DB: Load session messages + memory facts
    GW->>GW: Build context (system prompt + memory + history)
    GW-->>U: Completion response
    GW->>DB: Persist assistant message to session

    U->>GW: GET /v1/sessions/{id}/messages
    GW->>DB: Read messages for session
    GW-->>U: Message list + summaries
    end

    rect rgb(245,255,245)
    Note over U,DB: Memory Facts
    U->>GW: POST /v1/memory/facts (scope: project|session, content)
    GW->>DB: Insert memory fact
    GW-->>U: fact_id

    U->>GW: GET /v1/memory/facts?scope=project&project_id=X
    GW->>DB: Query facts by scope + owner
    GW-->>U: Fact list

    U->>GW: DELETE /v1/memory/facts/{id}
    GW->>DB: Remove fact
    GW-->>U: 204
    end
```

## 16) File Attachment Upload & Extraction

```mermaid
sequenceDiagram
    autonumber
    participant U as User
    participant GW as llm_port_api
    participant FS as FileStore (MinIO / Local)
    participant DOC as llm_port_docling
    participant DB as Postgres (llm_api)

    U->>GW: POST /v1/sessions/{session_id}/attachments (multipart file)
    GW->>FS: Store raw file
    FS-->>GW: storage_key

    alt Docling configured
      GW->>DOC: POST /convert (file bytes, format=markdown)
      DOC->>DOC: IBM Docling pipeline (pages, tables, images)
      DOC-->>GW: Extracted text + estimated token count
    else Local fallback
      GW->>GW: Basic text extraction
    end

    GW->>DB: Insert attachment record (file metadata + extracted text)
    GW-->>U: 201 attachment_id + metadata

    Note over U,GW: When next chat completion uses this session
    U->>GW: POST /v1/chat/completions (session_id)
    GW->>DB: Load attachments for session
    GW->>GW: Inject extracted text into context
    GW->>GW: Continue with normal completion flow
```

## 17) PII Screening Pipeline

```mermaid
sequenceDiagram
    autonumber
    participant C as API Client
    participant GW as llm_port_api
    participant PII as llm_port_pii
    participant LP as LLM Provider
    participant DB as Postgres (PII events)

    C->>GW: POST /v1/chat/completions
    GW->>GW: Resolve provider (cloud vs local)

    alt Cloud provider + PII policy enabled
      rect rgb(255,245,245)
      Note over GW,PII: Egress PII screening
      GW->>PII: POST /api/pii/sanitize (messages, mode=redact|tokenize)
      PII->>PII: Presidio NER analysis (names, emails, phones, SSNs, etc.)
      PII->>DB: Log PII event (entity types, counts)
      PII-->>GW: Sanitized messages + token map (if tokenize mode)
      end

      GW->>LP: Forward sanitized request
      LP-->>GW: Response

      alt Tokenize mode
        rect rgb(245,255,245)
        Note over GW,PII: Response detokenization
        GW->>PII: POST /api/pii/detokenize (response, token_map)
        PII-->>GW: Restored response with original PII values
        end
      end
    else Local provider or PII disabled
      GW->>LP: Forward request as-is
      LP-->>GW: Response
    end

    GW-->>C: Completion response
```

## 18) SSO Authentication Flow

```mermaid
sequenceDiagram
    autonumber
    participant U as User Browser
    participant FE as llm_port_frontend
    participant AUTH as llm_port_auth
    participant IDP as Identity Provider (Google/GitHub/etc.)
    participant BE as llm_port_backend

    U->>FE: Click SSO login button
    FE->>AUTH: GET /login/{provider} (redirect)
    AUTH->>AUTH: Build OAuth authorize URL (state + PKCE)
    AUTH-->>U: 302 Redirect to IdP

    U->>IDP: Authenticate + consent
    IDP-->>AUTH: GET /callback?code=XXX&state=YYY

    AUTH->>IDP: POST /token (exchange code for access_token)
    IDP-->>AUTH: access_token + id_token

    AUTH->>IDP: GET /userinfo (Bearer access_token)
    IDP-->>AUTH: email + profile claims

    AUTH->>AUTH: Sign JWT (email, account_id, exp=120s)
    AUTH-->>U: 302 Redirect to backend /api/auth/callback?token=JWT

    U->>BE: GET /api/auth/callback?token=JWT
    BE->>BE: Verify JWT signature
    BE->>BE: Find or create user account
    BE->>BE: Set httponly cookie (fapiauth)
    BE-->>U: 302 Redirect to /admin (authenticated)
```

## 19) Notification Delivery Pipeline

```mermaid
sequenceDiagram
    autonumber
    participant TRIGGER as Backend Event (password reset / 5xx alert / admin invite)
    participant BE as llm_port_backend
    participant DB as Postgres (notification_outbox)
    participant DISP as Background Dispatcher
    participant MAILER as llm_port_mailer
    participant SMTP as SMTP Provider

    TRIGGER->>BE: Event fires (e.g., user requests password reset)
    BE->>DB: INSERT notification_outbox (type, payload, status=pending)

    loop Dispatcher poll interval
      DISP->>DB: SELECT WHERE status=pending AND next_attempt_at <= now()
      DB-->>DISP: Pending notifications

      loop Each notification
        DISP->>DISP: Check dedup fingerprint + cooldown (30 min default)
        alt Not duplicated
          DISP->>MAILER: POST /internal/v1/messages/{type} (Bearer service token)
          MAILER->>MAILER: Render template (Jinja2)
          MAILER->>SMTP: Send email
          SMTP-->>MAILER: Delivery result
          MAILER-->>DISP: 200 OK / error

          alt Success
            DISP->>DB: UPDATE status=sent
          else Failure
            DISP->>DB: UPDATE status=retry, next_attempt_at += exponential backoff
          end
        else Duplicate suppressed
          DISP->>DB: UPDATE status=suppressed
        end
      end
    end
```

## 20) Module Enable / Disable Lifecycle

```mermaid
sequenceDiagram
    autonumber
    participant U as Admin User
    participant FE as llm_port_frontend (Settings)
    participant BE as llm_port_backend
    participant SET as SystemSettingsService
    participant REG as Module Registry
    participant DE as Docker Compose

    U->>FE: Toggle module (e.g., PII) on/off
    FE->>BE: PUT /api/admin/system/settings/values/{module_key}
    BE->>SET: Validate + persist setting

    SET->>REG: Notify module registry of state change
    REG->>REG: Run module sync callback (cleanup / init)

    alt Requires service lifecycle change
      SET->>DE: Start or stop Docker service (compose profile)
      DE-->>SET: Service started / stopped
    end

    SET-->>BE: Applied
    BE-->>FE: 200 + updated module state

    Note over FE: Frontend discovers module states
    FE->>BE: GET /api/v1/services (health probe)
    BE-->>FE: Module availability map
    FE->>FE: Show/hide module-dependent UI sections
```
