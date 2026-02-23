# llm.port

**llm.port** is a self-hosted **AI Gateway + Ops Console**: one OpenAI-compatible endpoint that
**routes, secures, and observes** requests across **local LLM runtimes** *and* **remote LLM providers**.

<img src="./images/dashboard.png">

<img src="./images/containers.png">

<img src="./images/models.png">

<img src="./images/logging.png">

<img src="./images/trace.png">

<img src="./images/api.png">

<img src="./images/rag_collectors.png">

## What we’re building

- **One endpoint for all apps**: OpenAI-compatible API (`/v1/*`)
- **Provider routing layer**: local runtimes (vLLM, llama.cpp, …) and remote providers (OpenAI, Azure, …)
- **Policies & controls**: auth (JWT), RBAC, quotas/rate limits, model allow-lists, request/response logging
- **Observability by default**: traces ([Langfuse](https://langfuse.com/)) + logs ([Loki](https://grafana.com/oss/loki/)/[Grafana](https://grafana.com/grafana)) wired through the gateway
- **Optional local runtime management**: deploy and operate containerized inference + model assets via [Huggingface](https://huggingface.co/)

## Features

- **OpenAI-compatible gateway**: `/v1/models`, `/v1/chat/completions`, `/v1/embeddings`
- **Centralized policy enforcement**: tenant-aware auth, model/provider restrictions, and rate controls
- **Unified observability**: live traces, request metadata, and operational dashboards
- **Ops console**: container/image/network/stack controls with RBAC and audit logs
- **System initialization flow**: guided setup for shared services and gateway configuration

Sequence reference:
- **Complete call sequence graphs**: [`call-sequence.md`](./call-sequence.md)

## Why it matters

You get **sovereign-by-default AI**: keep data on-prem when needed, use remote providers when allowed —
without changing your apps or losing governance and observability.

## Repositories

- `llm-port-frontend` — React admin console UI
- `llm-port-backend` — FastAPI control-plane + gateway API
- `llm-port-shared` — shared services (Grafana/Loki/Langfuse/Postgres, etc.)
- `scripts` — dev + operator scripts
