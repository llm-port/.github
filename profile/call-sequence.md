# llm.port - End-to-End Call Sequences

This document captures the main runtime and control-plane interactions across frontend, backend, gateway, and shared services.

## 1) OpenAI-Compatible Inference (Chat/Embeddings)

```mermaid
sequenceDiagram
    autonumber
    participant App as Client App / SDK
    participant GW as llm_port_api Gateway
    participant Redis as Redis (lease/limits)
    participant PG as Postgres (llm_api)
    participant LLM as Provider Runtime (vLLM/llama.cpp/Ollama)
    participant LF as Langfuse

    App->>GW: POST /v1/chat/completions or /v1/embeddings
    GW->>GW: Validate JWT + tenant_id claim
    GW->>PG: Load alias/pool/policy metadata
    GW->>Redis: Check RPM/TPM + acquire concurrency lease
    alt Lease acquired and policy allowed
        GW->>LLM: Forward OpenAI-compatible request
        LLM-->>GW: Response (stream/non-stream)
        GW->>LF: Trace/generation + usage + latency
        GW->>PG: Insert audit/request log row
        GW->>Redis: Release lease
        GW-->>App: OpenAI-compatible response (+x-request-id / trace id)
    else No capacity / policy denied / auth fail
        GW->>PG: Insert failure audit/request log row
        GW-->>App: OpenAI-style error envelope
    end
```

## 2) Admin Console - Container and Runtime Operations

```mermaid
sequenceDiagram
    autonumber
    participant User as Admin User (Browser)
    participant FE as airgap_frontend
    participant BE as airgap_backend (/api/admin, /api/llm)
    participant RBAC as RBAC + Root Mode Guards
    participant Docker as Docker Engine API
    participant Cont as Managed Containers
    participant Audit as Audit DB Tables

    User->>FE: Click Start/Stop/Restart or Create Runtime
    FE->>BE: POST /api/admin/containers/{id}/{action} or /api/llm/runtimes/*
    BE->>RBAC: Validate permission + root mode constraints
    alt Allowed
        BE->>Docker: Execute lifecycle/create request
        Docker->>Cont: Start/Stop/Restart/Create container
        Docker-->>BE: Updated container/runtime state
        BE->>Audit: Write allow event
        BE-->>FE: 2xx + latest state
    else Denied
        BE->>Audit: Write deny event
        BE-->>FE: 403/4xx
    end
```

## 3) System Settings Update + Immediate Apply

```mermaid
sequenceDiagram
    autonumber
    participant User as Admin User (Browser)
    participant FE as Settings UI (/admin/settings)
    participant BE as System API (/api/admin/system)
    participant Crypto as SettingsCrypto
    participant CFG as system_setting_* tables
    participant Job as system_apply_job(_event)
    participant Exec as Local Executor
    participant Compose as docker compose
    participant Shared as Shared Stack Services

    User->>FE: Edit setting key/value and Save
    FE->>BE: PUT /api/admin/system/settings/values/{key}
    BE->>BE: Validate schema + RBAC + root-mode if protected
    alt Secret setting
        BE->>Crypto: Encrypt value
        BE->>CFG: Upsert system_setting_secret
    else Non-secret setting
        BE->>CFG: Upsert system_setting_value
    end

    alt apply_scope = live_reload
        BE-->>FE: apply_status=success (no job)
    else apply_scope = service_restart or stack_recreate
        BE->>Job: Create apply job + start event
        BE->>Exec: Execute action list
        alt service_restart
            Exec->>Compose: Restart target service container(s)
        else stack_recreate
            Exec->>Compose: up -d --force-recreate (ordered services)
        end
        Compose->>Shared: Restart/Recreate services
        alt Apply success
            BE->>Job: Append success events + mark SUCCESS
            BE-->>FE: apply_status=success + job_id
        else Apply failure
            BE->>Job: Append failed event + start rollback
            BE->>Exec: Best-effort rollback attempt
            BE->>Job: Append rollback result + final FAILED/ROLLBACK_FAILED
            BE-->>FE: apply_status=failed + job_id
        end
    end
```

## 4) System Initialization Wizard

```mermaid
sequenceDiagram
    autonumber
    participant User as Admin User (Browser)
    participant FE as Wizard UI (/admin/settings?tab=system-init)
    participant BE as Wizard API (/api/admin/system/wizard/*)
    participant Settings as Shared Settings Service
    participant Apply as Immediate Apply Engine

    User->>FE: Open System Init Wizard
    FE->>BE: GET /api/admin/system/wizard/steps
    BE-->>FE: Step list (host, core-data, auth, gateway, observability, verify)

    loop Per wizard step
        User->>FE: Fill fields for current step
        FE->>BE: POST /api/admin/system/wizard/apply
        BE->>Settings: Update each key through common settings path
        Settings->>Apply: Trigger immediate apply where required
        Apply-->>BE: Per-key apply results
        BE-->>FE: Step apply summary
    end
```

## 5) LLM Tracing and Observability Pipeline

```mermaid
sequenceDiagram
    autonumber
    participant App as Client App
    participant GW as llm_port_api Gateway
    participant LF as Langfuse Web/Worker
    participant CH as ClickHouse
    participant PG as Postgres (Langfuse metadata)
    participant MinIO as MinIO
    participant BE as airgap_backend
    participant FE as airgap_frontend
    participant Loki as Loki
    participant Graf as Grafana

    App->>GW: Inference request
    GW->>LF: Send trace/generation events
    LF->>CH: Persist event/analytics data
    LF->>PG: Persist metadata/project refs
    LF->>MinIO: Persist media/blob objects (if any)

    FE->>BE: GET /api/llm/graph/traces + /stream
    BE->>PG: Read gateway request logs (trace model)
    PG-->>BE: Trace events
    BE-->>FE: Topology + trace stream (SSE)

    FE->>BE: GET /api/logs/*
    BE->>Loki: Query logs
    Loki-->>BE: Log results
    BE-->>FE: Filtered logs
    FE->>Graf: Open dashboards/panels
```

## 6) Agent Contract (Multi-Host Ready Path)

```mermaid
sequenceDiagram
    autonumber
    participant Agent as Infra Agent
    participant BE as airgap_backend (/api/admin/system/agents)
    participant Jobs as system_apply_job(_event)
    participant FE as Admin UI

    Agent->>BE: POST /agents/register
    BE->>Jobs: Persist/refresh agent state
    BE-->>Agent: Registration ack

    loop Heartbeats
        Agent->>BE: POST /agents/heartbeat
        BE-->>Agent: 200 OK
    end

    FE->>BE: POST /agents/{agent_id}/apply (signed bundle)
    BE->>Jobs: Create remote apply job + accepted event
    BE-->>FE: accepted=true, job_id

    FE->>BE: GET /agents/{agent_id}/jobs/{job_id}
    BE->>Jobs: Read apply job timeline
    BE-->>FE: Current status/events
```
