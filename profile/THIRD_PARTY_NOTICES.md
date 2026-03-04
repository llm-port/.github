# THIRD_PARTY_NOTICES

This document lists key third-party software that llm.port deploys, integrates, or depends on —
both Docker images (e.g., through docker-compose profiles such as `rag`, `pii`, `auth`, `mailer`, `docling`)
and library-level dependencies bundled into llm.port service images.

Each third-party component is licensed by its respective authors under its own license terms.
llm.port does not replace or modify those licenses.

> Practical tip: For compliance and reproducibility, prefer pinning Docker images to exact versions
> and/or digests (instead of floating tags like `:latest` or broad tags like `:7`).

---

## Core / Default stack

### PostgreSQL

- Images (example):
  - `postgres:18.1-bookworm` (llm_port_backend)
  - `postgres:16.3-bullseye` (llm_port_pii)
- Upstream license:
  - PostgreSQL License (permissive)

### PostgreSQL + pgvector

- Image (example):
  - `pgvector/pgvector:pg17` (llm_port_shared, llm_port_rag)
- Upstream licenses:
  - PostgreSQL: PostgreSQL License (permissive)
  - pgvector extension: PostgreSQL License (permissive)
- Notes:
  - This is a permissive licensing combo commonly accepted for commercial use.

### ClickHouse

- Image (example):
  - `clickhouse/clickhouse-server:latest`
- Upstream license:
  - Apache License 2.0

### Redis

- Images (example):
  - `redis:7` (llm_port_shared)
  - `bitnami/redis:6.2.5` (llm_port_api, llm_port_pii, llm_port_docling, llm_port_auth)
- Upstream licensing:
  - Redis licensing depends on the version pulled:
    - Redis <= 7.2 is BSD-3-Clause
    - Redis 7.4.x–7.8.x is dual-licensed RSALv2 or SSPLv1
    - Redis 8.0+ is tri-licensed RSALv2 / SSPLv1 / AGPLv3
  - Bitnami Redis 6.2.5 packages Redis 6.2 which is BSD-3-Clause.
- Notes:
  - Because `redis:7` is a moving tag, the effective license may change when the tag updates.
  - If you need BSD-only, pin to a known 7.2.x image or use `bitnami/redis:6.2.5`.

### MinIO (S3-compatible object storage)

- Image (example):
  - `cgr.dev/chainguard/minio:latest` (image provider may differ; license is about the upstream software)
- Upstream licensing:
  - MinIO states it is dual-licensed under GNU AGPLv3 and a commercial license.
- Notes:
  - AGPLv3 is a strong copyleft license; some organizations require legal review.
  - You can make MinIO optional by supporting "bring your own S3-compatible storage".

### RabbitMQ (AMQP broker)

- Images (example):
  - `rabbitmq:4.2.1-management-alpine` (llm_port_shared, llm_port_backend)
  - `rabbitmq:3.9.16-alpine` (llm_port_api, llm_port_pii, llm_port_rag)
- Upstream licensing:
  - RabbitMQ server is licensed under MPL 2.0 (file-level copyleft).
  - RabbitMQ management plugin is also MPL 2.0.

---

## Observability

### Grafana

- Image (example):
  - `grafana/grafana:11.5.2`
- Upstream licensing:
  - Grafana repository LICENSE is GNU AGPLv3.
- Notes:
  - AGPLv3 is strong copyleft; policies vary by organization.

### Grafana Loki

- Image (example):
  - `grafana/loki:3.0.0`
- Upstream licensing:
  - Loki repository LICENSE is GNU AGPLv3.

### Grafana Alloy

- Image (example):
  - `grafana/alloy:latest`
- Upstream licensing:
  - Apache License 2.0

### Grafana OTel LGTM (dev / testing stack)

- Image (example):
  - `grafana/otel-lgtm` (llm_port_backend OTLP dev compose)
- Upstream licensing:
  - Apache License 2.0 (OpenTelemetry Collector) + GNU AGPLv3 (Grafana, Loki) + Apache 2.0 (Tempo)
- Notes:
  - All-in-one dev image combining OTel Collector, Loki, Grafana, and Tempo. License varies per bundled component.

### OpenTelemetry Collector Contrib

- Image (example):
  - `otel/opentelemetry-collector-contrib:0.53.0` (OTLP dev composes across all modules)
- Upstream licensing:
  - Apache License 2.0

### Jaeger

- Image (example):
  - `jaegertracing/all-in-one:1.35` (OTLP dev composes across all modules)
- Upstream licensing:
  - Apache License 2.0

---

## LLM observability / tracing (Langfuse)

### Langfuse (web + worker)

- Images (example):
  - `langfuse/langfuse:3`
  - `langfuse/langfuse-worker:3`
- Upstream licensing:
  - Most of the repo is MIT ("MIT Expat"), with **Enterprise Edition** code under `ee/` (and related paths)
    covered by a separate license (`ee/LICENSE`).
- Notes:
  - Langfuse documents that **core features** are available in the OSS version,
    and **some enterprise features** require a license key when self-hosting.

---

## Optional module: Document Processing

### Docling (IBM)

- Images (example):
  - `ds4sd/docling-serve:latest` (llm_port_rag compose)
  - `llm-port-docling` (wrapper service built from this repo, bundles `docling` + `docling-core` Python packages)
- Upstream licensing:
  - Docling Python library: MIT
  - Docling-Serve Docker image: MIT
- Notes:
  - Docling also advises checking **model licenses** for any models you use alongside it.

---

## Key Python library dependencies

These libraries are bundled into llm.port service images at build time.

> No non-permissive (AGPL / GPL / SSPL) Python or npm library dependencies remain in the codebase.

### PII / NLP

| Library             | Service(s)   | License | Notes                                                                                                                                                |
| ------------------- | ------------ | ------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| presidio-analyzer   | llm_port_pii | MIT     | Microsoft Presidio PII detection engine                                                                                                              |
| presidio-anonymizer | llm_port_pii | MIT     | Microsoft Presidio PII anonymization engine                                                                                                          |
| spaCy               | llm_port_pii | MIT     | NLP framework used by Presidio. **Note:** spaCy NLP models may have their own licenses (e.g., `en_core_web_lg` is MIT, but other models may differ). |

### LLM ecosystem

| Library               | Service(s)       | License    | Notes                                |
| --------------------- | ---------------- | ---------- | ------------------------------------ |
| langfuse (Python SDK) | llm_port_api     | MIT        | LLM observability client             |
| huggingface-hub       | llm_port_backend | Apache 2.0 | HuggingFace model download & hub API |
| hf-xet                | llm_port_backend | Apache 2.0 | HuggingFace Xet storage transport    |

### Document processing

| Library      | Service(s)       | License | Notes                                        |
| ------------ | ---------------- | ------- | -------------------------------------------- |
| docling      | llm_port_docling | MIT     | IBM document conversion library              |
| docling-core | llm_port_docling | MIT     | IBM Docling core types & models              |
| pdfplumber   | llm_port_backend | MIT     | PDF text & table extraction (wraps pdfminer) |
| python-docx  | llm_port_backend | MIT     | DOCX file processing                         |
| python-pptx  | llm_port_backend | MIT     | PPTX file processing                         |
| openpyxl     | llm_port_backend | MIT     | Excel file processing                        |

### Core frameworks (used across most services)

| Library      | License          | Notes                    |
| ------------ | ---------------- | ------------------------ |
| FastAPI      | MIT              | Web framework            |
| Uvicorn      | BSD              | ASGI server              |
| Pydantic     | MIT              | Data validation          |
| SQLAlchemy   | MIT              | Database ORM             |
| asyncpg      | Apache 2.0       | PostgreSQL async driver  |
| Alembic      | MIT              | Database migrations      |
| httpx        | BSD              | Async HTTP client        |
| TaskIQ       | MIT              | Distributed task queue   |
| aio-pika     | Apache 2.0       | AMQP / RabbitMQ client   |
| cryptography | Apache 2.0 / BSD | Cryptographic primitives |

### Container & hardware management

| Library   | Service(s)       | License    | Notes                              |
| --------- | ---------------- | ---------- | ---------------------------------- |
| aiodocker | llm_port_backend | Apache 2.0 | Async Docker Engine API client     |
| pynvml    | llm_port_backend | BSD        | NVIDIA GPU monitoring (wraps NVML) |
| psutil    | llm_port_backend | BSD        | System/process monitoring          |

### Observability (Python libraries)

| Library                                                                      | License    | Notes                           |
| ---------------------------------------------------------------------------- | ---------- | ------------------------------- |
| opentelemetry-sdk, opentelemetry-api, opentelemetry-exporter-otlp-proto-grpc | Apache 2.0 | OpenTelemetry instrumentation   |
| opentelemetry-instrumentation-fastapi                                        | Apache 2.0 | FastAPI auto-instrumentation    |
| opentelemetry-instrumentation-sqlalchemy                                     | Apache 2.0 | SQLAlchemy auto-instrumentation |
| opentelemetry-instrumentation-aio-pika                                       | Apache 2.0 | RabbitMQ auto-instrumentation   |
| prometheus-client                                                            | Apache 2.0 | Prometheus metrics              |
| prometheus-fastapi-instrumentator                                            | ISC        | FastAPI Prometheus middleware   |
| sentry-sdk                                                                   | MIT        | Error tracking (optional)       |
| loguru                                                                       | MIT        | Structured logging              |

### Authentication

| Library       | Service(s)                      | License | Notes                              |
| ------------- | ------------------------------- | ------- | ---------------------------------- |
| fastapi-users | llm_port_backend                | MIT     | User management framework          |
| httpx-oauth   | llm_port_backend, llm_port_auth | MIT     | OAuth2 / OIDC provider integration |
| PyJWT         | llm_port_api, llm_port_auth     | MIT     | JWT token encoding/decoding        |

### CLI

| Library    | Service(s)   | License | Notes               |
| ---------- | ------------ | ------- | ------------------- |
| Click      | llm_port_cli | BSD     | CLI framework       |
| Rich       | llm_port_cli | MIT     | Terminal formatting |
| Textual    | llm_port_cli | MIT     | TUI framework       |
| InquirerPy | llm_port_cli | MIT     | Interactive prompts |
| Jinja2     | llm_port_cli | BSD     | Template engine     |

---

## Key frontend (npm) dependencies

| Library              | License | Notes                                                                       |
| -------------------- | ------- | --------------------------------------------------------------------------- |
| React, React DOM     | MIT     | UI framework                                                                |
| React Router         | MIT     | Client-side routing                                                         |
| MUI (Material UI)    | MIT     | Component library (`@mui/material`, `@mui/icons-material`, `@mui/x-charts`) |
| Emotion              | MIT     | CSS-in-JS (`@emotion/react`, `@emotion/styled`)                             |
| TanStack React Table | MIT     | Data table                                                                  |
| React Flow           | MIT     | Node-based graph/flow visualization                                         |
| dnd-kit              | MIT     | Drag-and-drop (`@dnd-kit/core`, `@dnd-kit/sortable`)                        |
| i18next              | MIT     | Internationalization (`i18next`, `react-i18next`)                           |
| Tailwind CSS         | MIT     | Utility-first CSS framework                                                 |

---

## llm.port components

Images built from this repository (e.g., `llm-port-api`, `llm-port-backend`, `llm-port-rag`,
`llm-port-pii`, `llm-port-mailer`, `llm-port-auth`, `llm-port-docling`, `llm-port-frontend`)
are licensed under the llm.port project license (see `LICENSE` in this repository).

---

## Where to find full license texts

For each upstream component, see the project's official LICENSE file and/or the vendor licensing pages.
This file is a notice and does not replace the original license terms.
