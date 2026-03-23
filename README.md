# 🤖 Model Context Protocol (MCP) — Research & Implementation Guide

> A production-ready guide to connecting LLMs with Java REST APIs using FastMCP and OpenAPI.

---

## 📋 Table of Contents

1. [What is MCP?](#1-what-is-mcp)
2. [Architecture Overview](#2-architecture-overview)
3. [Spec Provider Comparison](#3-spec-provider-comparison)
4. [Why OpenAPI?](#4-why-openapi)
5. [Dynamic Refresh Strategies](#5-dynamic-refresh-strategies)
6. [Production Design](#6-production-design)
7. [Demo & Quickstart](#7-demo--quickstart)
8. [Trade-offs & Next Steps](#8-trade-offs--next-steps)

---

## 1. What is MCP?

**Model Context Protocol (MCP)** is an open JSON-RPC standard that lets AI agents (like Claude or ChatGPT) connect to external tool servers in a **uniform, structured way**.

Think of MCP as a **universal plugin interface for LLMs** — the AI sees a list of typed tools it can call, without knowing the low-level HTTP/SQL/gRPC details underneath.

### Core Primitives

| Primitive | Description |
|-----------|-------------|
| 🔧 **Tools** | Executable functions (e.g. `searchForPets`) |
| 📦 **Resources** | Static data contexts (e.g. a file or DB content) |
| 💬 **Prompts** | Text templates (e.g. system messages or examples) |

### Protocol Layers

```
┌─────────────────────────────────────────────┐
│              DATA LAYER                      │
│  JSON-RPC protocol · Tools · Resources       │
│  Prompts · Notifications · Lifecycle         │
├─────────────────────────────────────────────┤
│            TRANSPORT LAYER                   │
│  STDIO (local) · Streamable HTTP (remote)    │
│  SSE (legacy streaming)                      │
└─────────────────────────────────────────────┘
```

### How FastMCP Fits In

[**FastMCP**](https://github.com/jlowin/fastmcp) is a Python framework that implements MCP. You write Python functions with type hints and docstrings — FastMCP auto-generates the JSON schemas, handles validation, and serves the full MCP lifecycle.

```python
from fastmcp import FastMCP

mcp = FastMCP("My API Server")

@mcp.tool
async def get_pet_by_id(pet_id: int) -> dict:
    """Retrieves a pet by its ID from the store."""
    ...
```

The key feature used in this project: **`OpenAPIProvider`** — it reads an OpenAPI spec and automatically converts every matching API endpoint into an MCP Tool. No manual code per endpoint needed.

---

## 2. Architecture Overview

```
┌───────────────┐        ┌─────────────────────────┐
│  User / LLM   │        │     MCP Server           │
│               │──────▶ │  FastMCP + OpenAPI       │
│  AI Agent     │ MCP    │  Provider                │
└───────────────┘ JSON-  │         │                │
                  RPC    │  Refresh Worker           │
                         └────────────┬────────────┘
                                      │ HTTP
                         ┌────────────▼────────────┐
                         │   Java REST Backend      │
                         │   (Spring Boot API)      │
                         └────────────┬────────────┘
                                      │
                         ┌────────────▼────────────┐
                         │  Database / CI/CD        │
                         │  Prometheus / Grafana    │
                         └─────────────────────────┘
```

**Flow:**
1. LLM Agent calls a tool via MCP JSON-RPC
2. FastMCP routes the call to the Java REST API via HTTP
3. Java API returns data → FastMCP returns it to the agent
4. A background worker periodically refreshes the OpenAPI spec to keep tools in sync

---

## 3. Spec Provider Comparison

We evaluated five approaches for auto-generating MCP tools from API specs:

| Dimension | OpenAPI | GraphQL | gRPC/Protobuf | AsyncAPI | Custom SDK |
|-----------|---------|---------|---------------|----------|------------|
| **Tool Generation** | ✅ Easy — FastMCP `OpenAPIProvider` ready | 🟡 Moderate — requires introspection mapping | 🔴 Hard — needs codegen + HTTP proxy | 🟡 Moderate — no native MCP support | 🔴 Manual — fully custom |
# Model Context Protocol (MCP) — Implementation Guide

This repository documents a practical approach to exposing Java REST APIs to language models via the Model Context Protocol (MCP) using FastMCP and OpenAPI. The goal is to provide a concise, production-oriented reference that engineering and data science teams can use to deploy a stable, secure MCP server.

## Overview

MCP is a lightweight JSON-RPC protocol that exposes backend capabilities to AI agents as typed "tools". In this project FastMCP (Python) is used to:

- Convert OpenAPI endpoints into MCP tools via an OpenAPI provider.
- Validate inputs using generated JSON schemas.
- Serve MCP over a streamable HTTP transport suitable for production.

This approach minimizes backend changes: the Java service continues to serve REST endpoints while FastMCP provides the MCP facade.

## Key Concepts

- Tools: callable operations with specified input/output schemas.
- Resources: contextual data served to clients as read-only objects.
- Prompts: templated messages or examples provided to clients.

## Why OpenAPI

OpenAPI is chosen because it:

- Integrates directly with Java (Spring) stacks through tools like springdoc.
- Is well supported by CI/CD workflows (linting, generator tooling).
- Provides rich, machine-readable schemas that FastMCP can convert into tools without manual wiring.

OpenAPI is pragmatic for teams that already maintain REST APIs. For larger APIs, consider curating or combining endpoints to avoid cognitive load for agents.

## Dynamic Refresh: Keeping Tools in Sync

APIs change. The MCP layer must reflect those changes. Common refresh patterns:

1. Polling loop: fetch the OpenAPI spec at a configured interval and rebuild the provider if the spec differs.
2. Hash detection: compute a stable hash of the spec and act only on changes to avoid unnecessary reloads.
3. Webhook trigger: CI/CD notifies the MCP server when a new API deploys; the server validates and refreshes.
4. Manual trigger: an administrative MCP tool or an authenticated HTTP route can force a refresh.

Implementation notes:

- Always validate the new spec before swapping providers.
- Use an async lock to perform atomic swaps so callers see a consistent tool set.
- Keep a reference to the previous provider for quick rollback if the new one fails.

## Production Considerations

Essentials for a production deployment:

- Containerize the MCP server (Python 3.11 slim). Use environment variables for configuration (spec URL, API base URL, refresh interval, secrets).
- Expose health and metrics endpoints for readiness/liveness and Prometheus integration.
- Require authentication on MCP endpoints and protect any refresh/webhook routes (secret header or mTLS).
- Implement basic rate limiting and observability (structured logs, call latencies, error counts).

Recommended environment variables:

- `OPENAPI_URL` — URL to the OpenAPI JSON
- `BASE_API_URL` — backend base URL used by the HTTP client
- `MCP_REFRESH_INTERVAL` — polling interval in seconds
- `MCP_SECRET` — secret for webhook-based refresh

## Security and Testing

- Protect incoming MCP calls with a bearer token or equivalent.
- Store API credentials in a secrets manager, not in the repo.
- Tests: unit tests for provider logic, integration tests for tool calls (using a mocked backend), and an end-to-end smoke test that verifies refresh behavior.

## Quickstart

1. Install dependencies (recommended in a virtual environment):

```bash
pip install fastmcp httpx
```

2. Configure environment variables (example):

```bash
export OPENAPI_URL="https://petstore3.swagger.io/api/v3/openapi.json"
export BASE_API_URL="https://petstore3.swagger.io"
export MCP_REFRESH_INTERVAL=300
export MCP_SECRET="replace-with-secret"
```

3. Start the server (example):

```bash
python server.py
```

4. Verify the server log shows the OpenAPI spec was loaded and tools were created. Use a FastMCP client to list and call tools.

## Example refresh flow (high level)

1. CI deploys a new Java API and updates OpenAPI JSON.
2. CI calls the MCP webhook (`/refresh`) including the shared secret.
3. MCP server fetches the updated spec, validates it, and atomically swaps the provider.
4. MCP sends a `tools/list_changed` notification to connected clients (if negotiated).

## Recommended Next Steps

- Curate tools: reduce noise by combining or filtering endpoints exposed to agents.
- Instrument metrics: track which tools are used and measure latencies and error rates.
- Build an agent demo: a small LangChain or agent example that demonstrates multi-step usage of the MCP tools.

## Summary

This repository demonstrates a low-friction path to expose REST APIs to LLM agents using FastMCP and OpenAPI. The approach emphasizes minimal backend changes, predictable refresh behavior, and production readiness through authentication, monitoring, and tests.

Last updated: March 2026
@mcp.tool
