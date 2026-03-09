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

    subgraph Modules["Optional Modules"]
        AUTH["llm_port_auth (SSO / OIDC)"]
        PII["llm_port_pii (Presidio)"]
        MAILER["llm_port_mailer (SMTP)"]
        DOCLING["llm_port_docling (IBM Docling)"]
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
        PGMAIN["Postgres (backend + auth)"]
        PGAPI["Postgres (gateway / sessions / memory)"]
        PGRAG["Postgres + pgvector (RAG)"]
        PGPII["Postgres (PII events)"]
        REDIS["Redis (rate limit / leases / cache)"]
        RMQ["RabbitMQ (Taskiq broker)"]
        MINIO["MinIO (raw snapshots/uploads)"]
        LANGFUSE["Langfuse (traces)"]
        LOKI["Loki + Alloy (logs)"]
        GRAF["Grafana (dashboards)"]
        DOCKER["Docker Engine / Compose"]
    end

    U1 --> FE
    FE -->|REST /api/*| BE
    FE -->|/chat UI| API
    U2 -->|OpenAI API /v1/*| API

    BE -->|Gateway admin/proxy calls| API
    BE -->|Internal RAG proxy /api/internal/*| RAGAPI
    BE -->|Auth provider proxy| AUTH
    BE -->|PII config/stats proxy| PII
    BE -->|Notification delivery| MAILER
    BE -->|Document conversion| DOCLING
    BE -->|Ops actions| DOCKER
    BE --> PGMAIN
    BE --> PGAPI
    BE --> LANGFUSE
    BE --> LOKI

    API -->|Route chat/embeddings| LOCAL
    API -->|Route chat/embeddings| REMOTE
    API -->|PII screening| PII
    API -->|Attachment extraction| DOCLING
    API --> PGAPI
    API --> REDIS
    API --> LANGFUSE

    AUTH -->|OAuth callback → JWT cookie| BE
    AUTH --> PGMAIN

    PII --> PGPII
    MAILER --> PGMAIN

    RAGAPI --> PGRAG
    RAGAPI --> MINIO
    RAGAPI --> DOCLING
    RAGAPI --> RMQ
    RAGW --> RMQ
    RAGW --> PGRAG
    RAGW --> MINIO
    RAGW --> DOCLING
    RAGS --> RMQ
    RAGS --> PGRAG

    GRAF --> LOKI
    GRAF --> PGMAIN
    FE -->|Open dashboards| GRAF
```

## Calling-Service Paths

1. **Admin operations path**
   `Browser → llm_port_frontend → llm_port_backend → Docker / Settings / Proxy targets`

2. **Application inference path**
   `App/SDK → llm_port_api → local runtime or remote provider → response`

3. **Chat completions path (console)**
   `Browser → llm_port_frontend → llm_port_backend (cookie→JWT proxy) → llm_port_api /v1/chat/completions → provider → SSE response`

4. **RAG query path**
   `Browser → llm_port_frontend → llm_port_backend /api/admin/rag/* → llm_port_rag /api/internal/knowledge/search`

5. **RAG publish path**
   `Browser upload → presigned MinIO upload → draft/publish API → RabbitMQ task → Taskiq worker → Docling extract → chunk → embed → pgvector index`

6. **PII screening path**
   `llm_port_api (pre-forward middleware) → llm_port_pii /analyze → redact/flag → continue or reject`

7. **SSO authentication path**
   `Browser → llm_port_auth /login/<provider> → IdP → llm_port_auth /callback → signed JWT → llm_port_backend (set cookie)`

8. **Notification delivery path**
   `llm_port_backend (outbox write) → dispatcher → llm_port_mailer /send → SMTP provider`

9. **Document processing path**
   `File upload → llm_port_docling /convert → IBM Docling pipeline → structured text → caller (RAG worker / gateway)`

10. **Observability path**
    `backend + gateway + rag → Loki/Langfuse → Grafana dashboards`
