# llm.port Architecture

This diagram shows the complete platform components and who calls whom.

```mermaid
flowchart LR
    subgraph Users["Users & Clients"]
        U1["Admin User (Browser)"]
        U2["Application Clients / SDKs"]
    end

    subgraph UI["Console Layer"]
        FE["llm_port_frontend (React)"]
    end

    subgraph Core["Control Plane"]
        BE["llm_port_backend (FastAPI)"]
    end

    subgraph Gateway["API Gateway Plane"]
        API["llm_port_api (OpenAI-compatible /v1/*)"]
    end

    subgraph RAG["RAG Plane (Internal)"]
        RAGAPI["llm_port_rag API (/api/internal/*)"]
        RAGW["Taskiq Worker"]
        RAGS["Scheduler"]
    end

    subgraph Runtime["Model Execution"]
        LOCAL["Local Runtimes (vLLM / llama.cpp / etc.)"]
        REMOTE["Remote Providers (OpenAI / Azure / etc.)"]
    end

    subgraph Shared["Shared Platform Services"]
        PGMAIN["Postgres (backend + rag metadata)"]
        PGAPI["Postgres (llm_api metadata/audit)"]
        REDIS["Redis (rate limit / leases / cache)"]
        RMQ["RabbitMQ (Taskiq broker)"]
        MINIO["MinIO (raw snapshots/uploads)"]
        LANGFUSE["Langfuse (traces)"]
        LOKI["Loki + Promtail (logs)"]
        GRAF["Grafana (dashboards)"]
        DOCKER["Docker Engine / Compose"]
    end

    U1 --> FE
    FE -->|REST /api/*| BE
    U2 -->|OpenAI API /v1/*| API

    BE -->|Gateway admin/proxy calls| API
    BE -->|Internal RAG proxy /api/internal/*| RAGAPI
    BE -->|Ops actions| DOCKER
    BE --> PGMAIN
    BE --> LANGFUSE
    BE --> LOKI

    API -->|Route chat/embeddings| LOCAL
    API -->|Route chat/embeddings| REMOTE
    API --> PGAPI
    API --> REDIS
    API --> LANGFUSE
    API --> LOKI

    RAGAPI --> PGMAIN
    RAGAPI --> MINIO
    RAGAPI --> RMQ
    RAGW --> RMQ
    RAGW --> PGMAIN
    RAGW --> MINIO
    RAGS --> RMQ
    RAGS --> PGMAIN

    GRAF --> LOKI
    GRAF --> PGMAIN
    FE -->|Open dashboards| GRAF
```

## Calling-Service Paths

1. **Admin operations path**
   `Browser -> llm_port_frontend -> llm_port_backend -> Docker/Settings/Proxy targets`

2. **Application inference path**
   `App/SDK -> llm_port_api -> local runtime or remote provider -> response`

3. **RAG query path**
   `Browser -> llm_port_frontend -> llm_port_backend /api/admin/rag/* -> llm_port_rag /api/internal/knowledge/search`

4. **RAG publish path**
   `Browser upload -> presigned MinIO upload -> draft/publish API -> RabbitMQ task -> Taskiq worker -> extract/chunk/embed/index`

5. **Observability path**
   `backend + gateway + rag -> Loki/Langfuse -> Grafana dashboards`

