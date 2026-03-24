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

## Spec providers — detailed comparison

This section covers the five spec/provider approaches considered for auto-generating MCP tools from existing services, and compares them on practical dimensions teams care about. Each provider summary includes when it is a good fit, key benefits, and important limitations.

### 1) OpenAPI (REST)

Summary
- OpenAPI (aka Swagger/OpenAPI) is a JSON/YAML specification for REST APIs that describes paths, parameters, request/response schemas, authentication schemes, examples, and more.

When to use
- You have a REST-first backend (e.g. Spring Boot controllers) and an established CI/CD process that already publishes Swagger/OpenAPI definitions.

Benefits
- Rich JSON Schema support for request and response bodies (nested objects, enums, examples).
- Mature tooling: code generators, validators, linters (Spectral), API documentation (Swagger UI, Redoc).
- Native support for describing auth (API keys, OAuth2 flows, JWT) and per-operation security.
- Language-neutral and widely supported by clients and servers.
- FastMCP `OpenAPIProvider` can directly convert endpoints into MCP tools with minimal glue code.

Limitations
- Large REST APIs can yield many small endpoints; raw auto-generation may produce a noisy tool surface for agents.
- No single-request aggregation semantics like GraphQL; agents may need to orchestrate multiple tool calls.

Example use-case
- Expose an existing Spring Boot service to LLMs with minimal backend changes; use CI to publish the spec and have MCP pull it.

Integration complexity: Low to moderate — typically one-off wiring in FastMCP and configuration of the HTTP client.

### 2) GraphQL

Summary
- GraphQL is a typed query language and runtime for APIs. A single endpoint accepts queries or mutations described by a strong schema (SDL). Introspection allows discovery of the schema at runtime.

When to use
- You control the API design and want flexible, client-driven queries that fetch exactly the fields needed in one request.

Benefits
- Clients can request and receive only the data they need in a single call, which reduces round trips and payload size for complex queries.
- Strong type system with precise field-level typing and introspection for discovery.

Limitations
- Requires changes or an additional layer if the backend is REST-first (adds implementation complexity).
- No standard way in the GraphQL spec to describe transport-level auth schemes; auth is usually handled by transport or custom directives.
- Less mature direct tooling for converting GraphQL operations into function-like MCP tools; would need a mapping layer to treat queries/mutations as tools.

Example use-case
- Greenfield APIs or cases where the client needs to assemble complex nested data from multiple resources in one call.

Integration complexity: Moderate to high — either you provide a GraphQL endpoint or add a gateway that maps GraphQL to REST endpoints.

### 3) gRPC / Protobuf

Summary
- gRPC uses protobuf-defined services and messages over HTTP/2 with binary encoding. It's ideal for high-performance, typed RPC between services.

When to use
- Service-to-service communication where performance and throughput matter, and the clients/servers can support gRPC natively.

Benefits
- Compact binary payloads and HTTP/2 multiplexing give excellent latency and throughput.
- Strong typing via .proto messages and official code generation for many languages.

Limitations
- Not a natural fit for LLM agents in the current ecosystem — most agent integrations expect HTTP/JSON or SDKs.
- Exposing gRPC to external LLM clients often requires an HTTP/JSON gateway (adds complexity).

Example use-case
- Internal microservices where low latency and high throughput are required; gRPC may sit behind a gateway for external/agent access.

Integration complexity: High if you need an HTTP layer; moderate if the stack already uses gRPC.

### 4) AsyncAPI (event-driven)

Summary
- AsyncAPI describes event-driven architectures (message brokers, topics, channels) and message payload schemas. It is suitable for pub/sub, stream, and event-driven systems.

When to use
- Your core interactions are asynchronous events rather than request/response, and you need to describe channels and message schemas.

Benefits
- Good for documenting producer/consumer contracts and message payloads; can be used to generate client scaffolding for publishers/subscribers.

Limitations
- Agents typically perform synchronous, request/response actions. Mapping event-driven flows into MCP tools is possible but often awkward.
- Tooling and direct MCP provider support for AsyncAPI is less widespread than OpenAPI.

Example use-case
- Streaming data pipelines, event ingestion systems, or services where LLMs should trigger or subscribe to domain events.

Integration complexity: Moderate — depends on whether you need synchronous wrappers around asynchronous channels.

### 5) Custom SDKs / Code-driven providers

Summary
- Directly using an existing SDK or language-native client to implement MCP tools: you wrap SDK calls in tool functions rather than generating tools from a separate spec.

When to use
- The backend exposes a mature SDK that already encapsulates business logic, or you need custom, curated tools that combine multiple lower-level APIs.

Benefits
- Full control: you decide tool granularity, names, and semantics. Can consolidate multiple REST calls into a single high-level tool.
- Can expose richer logic (retries, business rules, multi-step orchestration) that OpenAPI‑generated tools do not.

Limitations
- Manual work to implement and maintain tools; no single source of truth derived from an API spec.
- Harder to keep in sync with backend changes unless SDK is tightly versioned and maintained.

Example use-case
- High-value curated operations (e.g., “createCustomerProfileWithDefaults”) that call multiple backend endpoints and enforce business constraints.

Integration complexity: Variable — low if the SDK is available and simple to wrap, but maintenance burden is higher.

---

### Practical comparison (plain table)

The table below summarizes practical differences across dimensions that affect an MCP integration. This is a plain text table to keep it simple and printable.

| Dimension               | OpenAPI (REST)                      | GraphQL                           | gRPC / Protobuf                  | AsyncAPI (Events)                | Custom SDK / Code                |
|------------------------:|:-----------------------------------|:----------------------------------|:---------------------------------|:---------------------------------|:---------------------------------|
| Tool generation         | High (provider exists, low glue)    | Medium (needs mapping to tools)   | Low (requires gateway/wrappers)  | Medium (requires sync wrappers)  | Low (manual)                      |
| Schema expressiveness   | High (JSON Schema)                  | High (typed SDL)                  | High (protobuf types)            | High (message schemas)           | Varies (depends on SDK types)     |
| Auth described in spec  | Yes (securitySchemes)               | No (handled at transport)         | Transport-level (TLS/mTLS)      | Yes (some support)               | No standard (handled in code)     |
| Tooling ecosystem       | Very mature (generator, validator)  | Mature (Apollo, codegen)          | Mature (protoc toolchain)       | Moderate (generators exist)     | None standard (per-language)      |
| Runtime fit for LLM     | Good (straightforward mapping)      | Excellent (single-query fetches)  | Limited (not agent-native)      | Limited (async mismatch)        | Flexible (but manual)             |
| Integration effort      | Low–medium                          | Medium–high                       | High (if gateway needed)        | Medium                          | Medium–high (maintain manually)   |
| Best for                | REST-first stacks with CI/CD        | Flexible data needs, greenfield    | High-performance internal RPC   | Event-driven systems             | Curated, high-level operations     |

---

## Focused conclusion: why OpenAPI for this project

Given the constraints and goals of this project, OpenAPI is the most pragmatic choice. The decision is based on the following concrete observations:

1. Existing stack alignment: our backend is Java REST (Spring). OpenAPI integrates directly with Spring via common libraries (e.g. springdoc), which makes spec generation automatic and reliable.
2. Minimal backend change: exposing the existing OpenAPI JSON allows FastMCP to surface endpoints as tools without modifying the Java services.
3. Tooling and CI compatibility: OpenAPI fits into standard CI flows (linting, governance, client generation). This enables spec validation during build and a predictable release process.
4. FastMCP integration: there is an existing OpenAPIProvider that converts endpoints into MCP tools; this reduces engineering effort and risk.

Trade-offs and mitigations
- Trade-off: OpenAPI auto-generation can produce a large set of low-level tools that may overwhelm an agent. Mitigation: curate the tool surface (filter or rename routes, combine endpoints, or implement higher-level SDK-wrapped tools for complex operations).
- Trade-off: GraphQL would reduce round trips for complex queries but requires adding a new interface layer. Use GraphQL if you plan a long-term API redesign or if client-driven queries are central.

Conclusion statement
- For a Java REST-based codebase with existing OpenAPI output and a priority on rapid, low-risk integration with MCP, OpenAPI is the most focused and practical choice. It balances automation, observability, and minimal operational change; any limitations (tool noise or multi-call orchestration) can be mitigated by subsequent curation and the occasional hand-written tool wrapper.

## Why this matters for the repo

Selecting OpenAPI lets the team deliver an MCP surface quickly, validate agent interactions in staging, and iterate on tool curation based on real usage metrics. If future needs change (e.g., demand for efficient complex queries), the project can revisit GraphQL or hybrid approaches with a clear migration path.

## Next actions (repo)

- Add a short example showing how to exclude or rename routes in the OpenAPI provider (small code snippet or config).
- Add a brief guidance note on when to implement curated SDK-wrapped tools vs. rely on generated endpoints.

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
