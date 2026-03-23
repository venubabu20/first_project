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
| **Schema Expressiveness** | ✅ High — full JSON Schema | ✅ High — typed SDL | ✅ High — Protobuf | ✅ High — JSON Schema messages | ⚪ Unlimited but no standard |
| **Auth Support** | ✅ Native (OAuth2, API Keys, JWT) | 🟡 Ad-hoc, not in spec | ✅ TLS + token at transport | ✅ `securitySchemes` in spec | 🔴 Custom only |
| **Runtime Performance** | 🟡 Medium — HTTP/JSON overhead | 🟢 Good — single endpoint | ✅ Best — HTTP/2 + binary | ⚪ Variable — broker-dependent | ⚪ Depends on impl |
| **Tooling Ecosystem** | ✅ Very large — Swagger UI, codegen, etc. | 🟢 Growing — Apollo, GraphiQL | ✅ Large — official protoc tools | 🟡 Moderate — still niche | 🔴 None — DIY |
| **LLM Suitability** | 🟢 Good — each endpoint = 1 tool | ✅ Excellent — fetches exactly what's needed | 🔴 Limited — LLMs rarely call gRPC | 🔴 Poor — async/pub-sub mismatch | ⚪ Flexible but manual |

**Bottom Line:** OpenAPI is the best fit for LLM tooling on existing REST/Java stacks — broad ecosystem, easy codegen, and direct FastMCP support.

---

## 4. Why OpenAPI?

We chose OpenAPI for this project for three clear reasons:

### ✅ Native to Java REST
Our backend is Spring Boot. Libraries like `springdoc-openapi` auto-generate the Swagger spec from controller annotations — **no extra interface layer needed**. The spec stays in sync with the code.

### ✅ CI/CD Ready
OpenAPI fits directly into our existing pipelines:
- Spec validation and linting (Spectral)
- Client/stub generation (OpenAPI Generator, Maven plugin)
- Auto-publishing to SwaggerHub

### ✅ FastMCP Out-of-the-Box
```python
from fastmcp.contrib.openapi import OpenAPIProvider

provider = OpenAPIProvider(
    openapi_spec=spec_json,
    client=httpx.AsyncClient(base_url=BASE_API_URL)
)
mcp = FastMCP("Petstore MCP", providers=[provider])
```
Every new REST endpoint in the Java service **automatically becomes an MCP tool** after the next spec refresh — zero manual coding.

### ⚠️ Known Trade-off
Large APIs produce many fine-grained tools. This can overwhelm LLMs with too many options. Mitigation: use FastMCP route filters or combine related endpoints into higher-level tools later.

---

## 5. Dynamic Refresh Strategies

APIs evolve. A static MCP server becomes outdated. Here are four strategies to keep tools in sync:

### 5.1 Periodic Polling *(Simple — Recommended Start)*
Fetch the spec on a schedule and swap the provider if it changed.

```python
async def refresh_loop():
    while True:
        await asyncio.sleep(POLL_INTERVAL)  # e.g. 300 seconds
        try:
            new_spec = httpx.get(OPENAPI_URL).json()
            new_provider = OpenAPIProvider(openapi_spec=new_spec, client=api_client)
            async with lock:
                mcp.providers = [new_provider]
            log.info("Spec refreshed successfully")
        except Exception as e:
            log.error("Refresh failed — keeping old provider", exc_info=e)
```

**Failure handling:** On error, the old provider stays active. The server never crashes.

---

### 5.2 Hash-Change Detection *(Efficient)*
Only rebuild the provider when the spec has actually changed.

```python
last_hash = None

async def refresh_if_changed():
    global last_hash
    spec = httpx.get(OPENAPI_URL).json()
    new_hash = md5(json.dumps(spec, sort_keys=True).encode()).hexdigest()
    if new_hash == last_hash:
        return  # No change — skip rebuild
    last_hash = new_hash
    # Rebuild provider ...
```

**Benefit:** Avoids unnecessary CPU work and prevents spurious `tools/list_changed` notifications.

---

### 5.3 Webhook / CI Trigger *(Real-time)*
Let the CI/CD pipeline call the MCP server when a new API version is deployed.

```python
@mcp.custom_route("/refresh", methods=["POST"])
async def webhook_refresh(request):
    # Validate a secret header for auth
    new_provider = await build_provider()
    async with lock:
        mcp.providers = [new_provider]
    return JSONResponse({"status": "refreshed"})
```

**CI step example:**
```bash
curl -X POST https://mcp-server/refresh \
     -H "X-Refresh-Secret: $MCP_SECRET"
```

---

### 5.4 Manual MCP Tool Refresh *(Admin Use)*
Expose a special MCP tool that authorized clients can call to trigger a reload.

```python
@mcp.tool
async def refresh_openapi():
    """Admin: reload the OpenAPI spec and update available tools."""
    new_provider = await build_provider()
    async with lock:
        mcp.providers = [new_provider]
    return {"status": "ok", "tools_count": len(mcp.tools_list())}
```

> ⚠️ Protect this tool — hide it from the LLM's main tool list or require elevated permissions.

---

### Strategy Comparison

| Strategy | Latency | Complexity | Best For |
|----------|---------|------------|----------|
| Periodic Polling | Minutes | Low | Getting started |
| Hash Detection | Minutes (efficient) | Low | Reducing noise |
| Webhook Trigger | Seconds | Medium | CI/CD integrated teams |
| Manual Tool | On-demand | Low | Dev/admin workflows |

---

## 6. Production Design

### Docker Setup

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY server.py .
USER nobody
CMD ["python", "server.py"]
```

```bash
# requirements.txt
fastmcp
httpx
prometheus-client
```

### Environment Variables

| Variable | Description | Example |
|----------|-------------|---------|
| `OPENAPI_URL` | URL to the OpenAPI JSON spec | `https://api.example.com/openapi.json` |
| `BASE_API_URL` | Java REST API base URL | `https://api.example.com` |
| `MCP_REFRESH_INTERVAL` | Polling interval in seconds | `300` |
| `MCP_SECRET` | Auth secret for `/refresh` webhook | `s3cr3t` |
| `LOG_LEVEL` | Logging verbosity | `INFO` |

### Monitoring

- **Logs:** Structured JSON logs → ELK / CloudWatch
- **Metrics:** Prometheus via `/metrics` custom route
  - Tool call count & latency
  - Refresh success/failure rate
  - Error rate per tool
- **Health:** `/health` endpoint for Kubernetes readiness/liveness probes

### Security Checklist

- [ ] Bearer token auth on incoming MCP calls (`auth=BearerAuth(token)`)
- [ ] Secret header validation on `/refresh` webhook endpoint
- [ ] HTTPS only in production (TLS termination at load balancer)
- [ ] Rate limiting middleware to prevent abuse
- [ ] API credentials stored in env vars / secrets manager (never hardcoded)

### Testing Strategy

| Level | What to Test |
|-------|-------------|
| **Unit** | Individual tool functions, provider creation from mock spec |
| **Integration** | Full tool calls via FastMCP client against a local container |
| **End-to-End** | LLM agent using tools against live Petstore API |
| **Refresh** | Modify mock spec, verify new tools appear after refresh |

---

## 7. Demo & Quickstart

### Prerequisites

```bash
pip install fastmcp httpx
```

### Run the Server

```bash
export OPENAPI_URL="https://petstore3.swagger.io/api/v3/openapi.json"
export BASE_API_URL="https://petstore3.swagger.io"
export MCP_REFRESH_INTERVAL=300

python server.py
```

Expected output:
```
INFO  Connected to Java API at https://petstore3.swagger.io
INFO  Loaded 20 tools from OpenAPI spec
INFO  MCP server running on http://0.0.0.0:8000
```

### Run the Client

```bash
python client.py
```

Expected output:
```
Connected to MCP Server
Available tools (20):
  • findPetsByStatus   — Finds Pets by status
  • getPetById         — Find pet by ID
  • addPet             — Add a new pet to the store
  • updatePet          — Update an existing pet
  ...
```

### Call a Tool

```python
result = await client.call_tool("findPetsByStatus", {"status": "available"})
print(result)
# [{"id": 1, "name": "Buddy", "status": "available"}, ...]
```

### Test Dynamic Refresh

1. Add a new endpoint to your OpenAPI JSON (or wait for the next polling cycle)
2. Watch the server logs: `INFO  Spec refreshed — 21 tools loaded`
3. Re-list tools on the client: the new tool appears — **no restart needed**

---

## 8. Trade-offs & Next Steps

### Trade-offs

| Decision | Pro | Con |
|----------|-----|-----|
| OpenAPI auto-generation | Zero manual coding per endpoint | Many fine-grained tools can overwhelm LLMs |
| Polling refresh | Simple, self-contained | Up to N-minute delay for spec changes |
| FastMCP OpenAPIProvider | Rapid integration | Tied to Python server for MCP layer |

### Next Steps

- 🔧 **Tool Curation** — Use FastMCP Transforms to filter, rename, or combine tools to reduce noise
- 🔐 **OAuth2** — Integrate OAuth2 flows for Java API calls requiring delegated auth
- 📊 **Metrics** — Add Prometheus instrumentation with Grafana dashboards
- 🤖 **Agent Demo** — Build a LangChain or Claude agent that uses this MCP to answer natural language queries
- 📉 **Usage Analytics** — Log which tools are called (and never called) to optimize the tool surface

---

## Summary

| What | Detail |
|------|--------|
| **Protocol** | MCP (Model Context Protocol) — JSON-RPC over HTTP |
| **Framework** | FastMCP (Python) |
| **Tool Source** | OpenAPI spec from Java Spring Boot backend |
| **Refresh** | Periodic polling + hash detection + webhook support |
| **Deployment** | Docker · Kubernetes · Prometheus monitoring |
| **Auth** | Bearer token (MCP) + API key (backend) |

> This setup exposes a full Java REST API to LLM agents **without changing a single line of backend code** — just point FastMCP at your Swagger URL.

---

*Last updated: March 2026*
