# llm.port

**llm.port** is a self-hosted **all-in-one LLM platform** that combines:
- an OpenAI-compatible gateway for app traffic
- control-plane operations for model servers and runtime infrastructure
- an internal RAG subsystem for ingestion, indexing, and governed retrieval

It **routes, secures, and observes** traffic across **local LLM runtimes** *and* **remote providers**, while giving teams a single place to manage LLM services end-to-end.


<img src="./images/dashboard.png">

<img src="./images/containers.png">

<img src="./images/models.png">

<img src="./images/logging.png">

<img src="./images/trace.png">

<img src="./images/api.png">

<img src="./images/rag_collectors.png">

---

## What it does

### ✅ Today
- **One endpoint for all apps**: OpenAI-compatible API (`/v1/*`)
- **Provider routing layer**: local runtimes (vLLM, llama.cpp, …) and remote providers (OpenAI, Azure, …)
- **Policies & controls**: auth (JWT), RBAC, quotas/rate limits, model allow-lists, request/response logging
- **Observability by default**: traces (Langfuse) + logs (Loki/Grafana) wired through the gateway
- **Ops console**: container/image/network/stack controls with RBAC and audit logs
- **System initialization flow**: guided setup for shared services and gateway configuration
- **Optional local runtime management**: deploy and operate containerized inference + model assets via Hugging Face
- **Internal RAG subsystem**: collectors, ACL-aware multi-tenant retrieval, runtime embedding config, and container upload/draft/publish workflows

### 🚧 In progress
- **Workspace/tenant retrieval scope**: strict isolation and policy-driven access

### 🧭 Planned
- More runtimes (ex: TensorRT-LLM, etc.)
- More providers (ex: additional managed APIs)
- Model rollout workflows (staged deploy, canary, rollback)
- Fine-grained cost controls and usage analytics

---

## Features

- **OpenAI-compatible gateway**: `/v1/models`, `/v1/chat/completions`, `/v1/embeddings`
- **Centralized policy enforcement**: tenant-aware auth, model/provider restrictions, and rate controls
- **Unified observability**: live traces, request metadata, and operational dashboards
- **Ops console**: manage images/containers/stacks with RBAC + audit logs
- **Guided setup**: bootstrap shared services + connect providers/runtimes

Sequence reference:
- **Complete call sequence graphs**: [`call-sequence.md`](./call-sequence.md)
- **Complete block architecture**: [`architecture.md`](./architecture.md)

---

## RAG (Retrieval)

The RAG portion runs as an internal service (`llm_port_rag`) with a pluggable connector/indexing layer:

- **Collectors**: pluggable source connectors (local folder/SMB stand-in, SharePoint stub, extensible registry)
- **Pipelines**: raw snapshot -> extraction -> normalization -> chunking -> embedding -> indexing
- **Scopes**: tenant + workspace boundaries with ACL principals filtering
- **Embedding model control**: configured centrally in **llm.port** and pushed to RAG runtime config
- **Knowledge containers**: N-level container tree with direct uploads to MinIO via presigned URLs
- **Draft/Publish model**: users stage operations and publish now or scheduled time, processed asynchronously via Taskiq + RabbitMQ

> Goal: make retrieval “just another policy-controlled capability” behind the same gateway and observability stack.

---

## Why it matters

You get **sovereign-by-default AI**: keep data on-prem when needed, use remote providers when allowed —
without changing your apps or losing governance and observability.

---

## Repository layout

- `llm-port-frontend` — React admin console UI
- `llm-port-backend` — FastAPI control-plane + gateway API
- `llm-port-shared` — shared services (Grafana/Loki/Langfuse/Postgres, etc.)
- `llm-port-api` — OpenAI specific V1 Api service
- `llm-port-rag` — RAG Service
- `scripts` — dev + operator scripts

> Visibility note: the org is public, but some repos may be private temporarily while we finalizing and cleaning things up.
---

## Getting started (preview)

Until the public quickstart is finalized, the intended flow is:

1) Start shared services (Grafana/Loki/Langfuse/Postgres)
2) Start the gateway/control-plane
3) Add providers (remote) and/or runtimes (local)
4) Point your apps to the OpenAI-compatible endpoint

If you want early access or to be a design partner, open an issue/discussion in this repo.

---

## Contributing

Contributions are welcome. If you’d like to help early, feel free to get in contact.

---

## License

Apache 2.0
