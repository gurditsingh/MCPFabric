# Enterprise MCP Gateway Lab — Analysis & Architecture

**Status:** Architecture analysis / implementation kickoff  
**Purpose:** Provide a concrete, end-to-end project specification that can be handed to Codex for implementation.  
**Scope of this document:** MCP Gateway only. Auto-discovery / AutoCrawler is intentionally deferred.

---

## 1. Executive decision

The proposed project is feasible and is a strong practical way to learn the architecture described by Uber.

The project should **not** treat Ollama as the MCP server. The clean separation is:

- **Ollama** = local LLM that decides when a tool should be called.
- **Agent/Orchestrator** = MCP client + Ollama client; translates MCP tool definitions into Ollama function tools and runs the agent loop.
- **MCP Gateway** = one remote MCP server exposed over Streamable HTTP. It lists curated tools, enforces policies, routes calls, invokes backend APIs, transforms results, and writes audit records.
- **Business API** = ordinary REST/OpenAPI application containing 10–20 endpoints.
- **PostgreSQL** = stores both application data and MCP control-plane metadata.
- **Docker Compose** = starts the local environment.

This gives us a realistic enterprise topology while remaining small enough to build locally.

---

## 2. Learning objective

The project must prove the following architecture:

```text
                         USER
                          |
                          v
                  +----------------+
                  | Agent / CLI    |
                  |                |
                  | Ollama client  |
                  | MCP client     |
                  +-------+--------+
                          |
              +-----------+-----------+
              |                       |
              v                       v
        +-----------+          +--------------+
        |  Ollama   |          | MCP Gateway  |
        | local LLM |          | /mcp         |
        +-----------+          +------+-------+
                                      |
                          +-----------+------------+
                          |                        |
                          v                        v
                   Control Plane              Data Plane
                   Registry/Policy            Runtime Router
                          |                        |
                          +-----------+------------+
                                      |
                                      v
                              +---------------+
                              | DataOps REST  |
                              | API / OpenAPI |
                              +-------+-------+
                                      |
                                      v
                                PostgreSQL
```

The most important outcome is to see that the backend REST application does not need to become MCP-aware.

The agent uses MCP.  
The gateway translates MCP tool calls to existing REST API calls.

---

## 3. Core use case

Build a small **DataOps / ETL Operations Platform**.

The platform manages:

- pipelines,
- pipeline executions,
- execution logs,
- data-quality failures,
- datasets,
- lineage,
- operational incidents.

A representative user question is:

> Why did the `customer_daily_load` pipeline fail? Check the execution, logs and data-quality failures. If the failure is retryable, rerun it and create an incident if the rerun fails.

This one request forces the agent to invoke several independent tools and demonstrates why a gateway is useful.

A possible agent sequence is:

```text
User request
   |
   v
get_pipeline()
   |
   v
get_run_status()
   |
   v
get_run_logs()
   |
   v
get_dq_failures()
   |
   +---- retryable? ---- no ----> explain failure
   |
  yes
   |
   v
rerun_failed_run()
   |
   v
get_run_status()
   |
   +---- failed ----> create_incident()
```

---

## 4. Technology choices

| Layer | Proposed technology | Reason |
|---|---|---|
| Business API | Python + FastAPI | Simple REST development and automatic OpenAPI generation |
| Database | PostgreSQL | Real relational persistence and useful for both app + control-plane data |
| ORM | SQLAlchemy 2.x | Mature async/sync database integration |
| Migrations | Alembic | Version database schema |
| MCP Gateway | Official MCP Python SDK | Native MCP protocol implementation |
| MCP transport | Streamable HTTP | Appropriate remote/deployed MCP transport |
| HTTP client | httpx | Async backend API invocation |
| Validation | JSON Schema / Pydantic | Validate tool inputs and responses |
| Local LLM | Ollama | Fully local inference and tool/function calling |
| Initial model | Configurable; start with a tool-capable model such as Qwen3 | Ollama officially demonstrates tool calling with Qwen3 |
| Agent | Python service/CLI | Bridges Ollama tool calling with MCP |
| Container runtime | Docker Compose | One-command multi-service local environment |
| Python dependency management | uv | Fast, reproducible local developer workflow |
| Tests | pytest + pytest-asyncio | Unit/integration coverage |

---

## 5. Important Ollama clarification

### Is “MCP server with Ollama” feasible?

Yes, but the wording should be corrected.

Ollama itself should **not** be the MCP server.

The desired flow is:

```text
                   tools/list
Agent --------------------------------> MCP Gateway
  |
  | convert MCP Tool definitions
  | into Ollama function-tool schemas
  v
Ollama
  |
  | chooses:
  | get_run_logs(run_id=123)
  v
Agent
  |
  | tools/call
  v
MCP Gateway
  |
  v
REST API
```

The Agent is the bridge.

Ollama supports function/tool calling and can return structured `tool_calls`. The agent executes those calls through the MCP client, appends the tool results to the conversation, and calls Ollama again.

This separation is important because it keeps the gateway model-independent. Later the same MCP Gateway could be consumed by Claude, Codex, another MCP client, or a different local LLM without changing the backend APIs.

---

## 6. Business API scope

Implement approximately 18 REST operations. FastAPI should expose OpenAPI at `/openapi.json` and interactive docs at `/docs`.

| # | Method | Path | Purpose | MCP exposure |
|---:|---|---|---|---|
| 1 | GET | `/pipelines` | List pipelines | Yes |
| 2 | POST | `/pipelines` | Create pipeline | No initially |
| 3 | GET | `/pipelines/{pipeline_id}` | Get pipeline | Yes |
| 4 | PATCH | `/pipelines/{pipeline_id}` | Update pipeline | No initially |
| 5 | DELETE | `/pipelines/{pipeline_id}` | Delete pipeline | No |
| 6 | POST | `/pipelines/{pipeline_id}/runs` | Start a run | Optional |
| 7 | GET | `/runs/{run_id}` | Get execution status/details | Yes |
| 8 | GET | `/runs/{run_id}/logs` | Get execution logs | Yes |
| 9 | GET | `/runs/{run_id}/dq-failures` | Get DQ failures | Yes |
| 10 | POST | `/runs/{run_id}/rerun` | Rerun failed execution | Yes, write/high-risk |
| 11 | POST | `/runs/{run_id}/cancel` | Cancel running execution | Yes, write/high-risk |
| 12 | GET | `/datasets` | List datasets | Yes |
| 13 | GET | `/datasets/{dataset_id}` | Get dataset metadata | Yes |
| 14 | GET | `/datasets/{dataset_id}/lineage` | Get upstream/downstream lineage | Yes |
| 15 | GET | `/incidents` | List incidents | Yes |
| 16 | POST | `/incidents` | Create incident | Yes, write |
| 17 | GET | `/incidents/{incident_id}` | Get incident | Yes |
| 18 | PATCH | `/incidents/{incident_id}` | Update incident | Optional |
| 19 | GET | `/health` | Service health | Not an LLM tool |

The key architecture lesson is that the **REST API surface and MCP tool surface are intentionally different**.

We should not expose every API operation merely because it exists.

---

## 7. Initial MCP tool catalog

Auto-discovery is out of scope. Tools are manually curated and registered.

Recommended first catalog:

| MCP tool | Backend operation | Risk | Notes |
|---|---|---|---|
| `list_pipelines` | `GET /pipelines` | Read | Direct mapping |
| `get_pipeline` | `GET /pipelines/{id}` | Read | Direct mapping |
| `get_run_status` | `GET /runs/{id}` | Read | Direct mapping |
| `get_run_logs` | `GET /runs/{id}/logs` | Read | Direct mapping |
| `get_dq_failures` | `GET /runs/{id}/dq-failures` | Read | Direct mapping |
| `rerun_failed_run` | `POST /runs/{id}/rerun` | Write | Confirmation/policy required |
| `cancel_run` | `POST /runs/{id}/cancel` | Write | Confirmation/policy required |
| `list_datasets` | `GET /datasets` | Read | Direct mapping |
| `get_dataset` | `GET /datasets/{id}` | Read | Direct mapping |
| `get_dataset_lineage` | `GET /datasets/{id}/lineage` | Read | Direct mapping |
| `list_incidents` | `GET /incidents` | Read | Direct mapping |
| `create_incident` | `POST /incidents` | Write | Audited |
| `get_incident` | `GET /incidents/{id}` | Read | Direct mapping |

`delete_pipeline` is intentionally not exposed. This demonstrates:

```text
API availability != MCP availability
```

---

## 8. OpenAPI usage in Phase 1

OpenAPI is important from the beginning, but **we will not yet auto-discover tools from it**.

FastAPI produces an OpenAPI document for the business service.

Every backend operation should have a stable `operationId`, for example:

```text
getPipeline
getRunStatus
getRunLogs
getDQFailures
rerunRun
cancelRun
createIncident
```

The MCP Registry will manually reference these operation IDs.

Example logical mapping:

```text
MCP tool:
    get_run_logs

OpenAPI operation:
    getRunLogs

Backend:
    dataops-api

Method:
    GET

Path:
    /runs/{run_id}/logs
```

Why keep the `operationId` now?

Because in the future AutoCrawler can inspect `/openapi.json` and create candidate tool definitions using the same stable identifiers. Phase 1 therefore creates the correct foundation without implementing discovery prematurely.

---

## 9. Control Plane design

The control plane is configuration and governance.

For this project, control-plane metadata should be persisted in PostgreSQL under an `mcp` schema.

Recommended tables:

| Table | Purpose |
|---|---|
| `mcp.servers` | Logical backend/service registrations |
| `mcp.tools` | MCP tool name, description, schemas, status, owner |
| `mcp.routes` | Maps a tool to backend protocol/method/path/operationId |
| `mcp.policies` | Read/write classification, allowed roles, confirmation requirement |
| `mcp.tool_versions` | Optional tool-definition history |
| `mcp.audit_events` | Runtime tool-call audit trail |

A representative `mcp.tools` record should conceptually contain:

```text
tool_name             = rerun_failed_run
description           = Rerun a failed ETL execution...
input_schema          = JSON Schema
output_schema         = JSON Schema
owner                 = data-platform
enabled               = true
risk                  = write
confirmation_required = true
```

The route record contains:

```text
service       = dataops-api
protocol      = http
http_method   = POST
path_template = /runs/{run_id}/rerun
operation_id  = rerunRun
```

This lets us prove a critical gateway behavior:

> A tool can be disabled or remapped through control-plane data without changing agent code and without rebuilding the business API.

---

## 10. Control Plane management interface

Do not build a large UI in Phase 1.

Expose a small internal admin REST interface from the Gateway or a separate management router:

```text
GET    /admin/tools
GET    /admin/tools/{tool_name}
POST   /admin/tools
PATCH  /admin/tools/{tool_name}
POST   /admin/tools/{tool_name}/enable
POST   /admin/tools/{tool_name}/disable
GET    /admin/audit
```

These are **not MCP tools**.

They are management APIs for configuring the gateway.

This makes the distinction clear:

```text
/admin/* = control plane

/mcp     = data plane
```

---

## 11. Data Plane design

The MCP Gateway exposes one MCP endpoint:

```text
http://mcp-gateway:8000/mcp
```

The official MCP SDK's low-level server API is preferred here because tool definitions come from the registry rather than Python decorators.

The gateway must implement two core dynamic operations:

```text
tools/list
tools/call
```

### `tools/list`

Runtime behavior:

```text
MCP client
   |
   | tools/list
   v
Gateway
   |
   v
Query enabled tools
   |
   v
Convert registry records
to MCP Tool objects
   |
   v
Return tools
```

### `tools/call`

Runtime behavior:

```text
MCP client
   |
   | tools/call(get_run_logs)
   v
Gateway
   |
   +--> find tool
   +--> verify enabled
   +--> validate arguments
   +--> evaluate policy
   +--> find route
   +--> invoke adapter
   +--> normalize response
   +--> audit call
   |
   v
MCP result
```

This is the heart of the project.

---

## 12. Backend adapter abstraction

Although Phase 1 has an HTTP backend, the gateway should not hard-code HTTP logic throughout the router.

Define a logical adapter contract:

```text
BackendAdapter
    invoke(route, arguments, context) -> GatewayResult
```

First implementation:

```text
HttpBackendAdapter
```

Later additions can include:

```text
GrpcBackendAdapter
NativeMcpBackendAdapter
LambdaBackendAdapter
```

The gateway router therefore does not need to understand each protocol.

This mirrors the enterprise gateway pattern where MCP remains stable while backend protocols vary.

---

## 13. Tool parameter mapping

A tool invocation may contain:

```text
{
  "run_id": 101
}
```

The route may contain:

```text
POST /runs/{run_id}/rerun
```

The HTTP adapter needs deterministic mapping rules for:

- path parameters,
- query parameters,
- request body,
- headers.

Do not infer these mappings with an LLM.

Store the mapping explicitly in control-plane metadata.

Example:

```text
run_id -> path.run_id
reason -> body.reason
include_history -> query.include_history
```

This keeps routing deterministic, testable, and secure.

---

## 14. Policy model

We do not need full OAuth/RBAC in the first implementation, but the architecture should include policy evaluation.

Use simulated caller context:

```text
X-User: alice
X-Role: developer
```

or an equivalent local test identity passed by the agent.

Example policies:

| Tool | Allowed roles | Confirmation |
|---|---|---|
| `get_run_status` | developer, operator | No |
| `get_run_logs` | developer, operator | No |
| `get_dq_failures` | developer, operator | No |
| `rerun_failed_run` | operator | Yes |
| `cancel_run` | operator | Yes |
| `create_incident` | developer, operator | No |

The policy layer should be an interface so a future implementation can replace the local role check with OAuth/JWT/IAM without changing the core gateway router.

---

## 15. Write-tool confirmation

MCP tool metadata can indicate that an operation has side effects, but the application still needs an explicit policy for writes.

For the learning project, write tools should accept an explicit confirmation flag or execution token from the Agent layer.

Example behavioral contract:

```text
rerun_failed_run

risk = write
confirmation_required = true
```

If confirmation is absent:

```text
Gateway -> DENIED / confirmation required
```

This is intentionally implemented in the data plane, while the rule itself is stored in the control plane.

That demonstrates:

```text
Control Plane defines policy
Data Plane enforces policy
```

---

## 16. Application database model

Use PostgreSQL with two logical schemas.

### `app` schema

| Table | Purpose |
|---|---|
| `app.pipelines` | Pipeline definitions |
| `app.pipeline_runs` | Run status/history |
| `app.run_logs` | Execution log entries |
| `app.dq_failures` | Data-quality failures |
| `app.datasets` | Dataset metadata |
| `app.lineage_edges` | Dataset lineage |
| `app.incidents` | Operational incidents |

Seed realistic data so the LLM has something to reason over.

Examples should include:

- successful runs,
- a failed run due to DQ errors,
- a retryable infrastructure failure,
- a currently running execution,
- a lineage graph,
- open and resolved incidents.

### `mcp` schema

Contains the gateway control-plane tables described above.

This allows a single PostgreSQL container while preserving the conceptual separation.

---

## 17. Agent + Ollama design

The Agent service should be deliberately thin.

Responsibilities:

```text
1. Connect to MCP Gateway.
2. Call tools/list.
3. Convert MCP tool schemas into Ollama function tools.
4. Send user prompt + tools to Ollama.
5. Read Ollama tool_calls.
6. Invoke those tools with MCP tools/call.
7. Append tool results to conversation.
8. Repeat until Ollama returns no tool calls.
9. Return final answer to user.
```

The Agent should not know backend URLs such as `/runs/{id}/logs`.

Only the Gateway knows backend routing.

That gives us this decoupling:

```text
Ollama
   |
Agent
   |
MCP
   |
Gateway
   |
HTTP
   |
DataOps API
```

---

## 18. Ollama model strategy

Do not make the project depend on one model.

Use environment configuration:

```text
OLLAMA_MODEL=<model>
```

The initial model must support tool/function calling reliably.

A reasonable starting family is Qwen3 because Ollama's official tool-calling documentation demonstrates Qwen3 for single, parallel, and multi-turn tool invocation.

The model can later be swapped without changing the MCP Gateway.

The key test is not generic chat quality. It is:

- choosing the correct tool,
- producing valid tool arguments,
- handling tool results,
- running a multi-step tool loop.

---

## 19. Docker topology

The full local environment should be startable through Docker Compose.

Logical services:

```text
+------------------------------------------------------+
|                    Docker Compose                    |
|                                                      |
|  +------------+       +---------------------------+  |
|  | PostgreSQL |<----->| DataOps API               |  |
|  |            |       | :8080                     |  |
|  +-----+------+       +-------------+-------------+  |
|        ^                            ^                |
|        |                            | HTTP           |
|        |                            |                |
|  +-----+----------------------------+-------------+  |
|  | MCP Gateway :8000 /mcp                        |  |
|  +---------------------------+--------------------+  |
|                              ^                       |
|                              | MCP                   |
|                              |                       |
|                    +---------+----------+            |
|                    | Agent / CLI         |            |
|                    +---------+----------+            |
|                              |                       |
|                              | Ollama API            |
|                              v                       |
|                    +--------------------+            |
|                    | Ollama :11434      |            |
|                    +--------------------+            |
+------------------------------------------------------+
```

Recommended Compose services:

| Service | Role |
|---|---|
| `postgres` | App + MCP metadata |
| `dataops-api` | REST/OpenAPI backend |
| `mcp-gateway` | Control + data plane |
| `ollama` | Local LLM API |
| `model-init` | Optional one-shot model pull |
| `agent` | Interactive agent/API/CLI |

A single command should start the stack:

```text
docker compose up --build
```

Ollama's official Docker image supports CPU execution. GPU configuration can be an optional profile rather than a prerequisite.

---

## 20. Development ergonomics

Provide these developer entry points:

```text
Business API Swagger:
http://localhost:8080/docs

Business OpenAPI:
http://localhost:8080/openapi.json

MCP Gateway:
http://localhost:8000/mcp

Gateway Admin API:
http://localhost:8000/admin/...

Ollama:
http://localhost:11434
```

The project should include:

```text
.env.example
compose.yaml
Makefile or task runner
README.md
docs/
```

No secrets should be committed.

---

## 21. Suggested repository structure

```text
enterprise-mcp-gateway-lab/
|
|-- compose.yaml
|-- .env.example
|-- Makefile
|-- README.md
|
|-- docs/
|   |-- analysis.md
|   |-- architecture.md
|   `-- api-tool-mapping.md
|
|-- services/
|   |
|   |-- dataops-api/
|   |   |-- pyproject.toml
|   |   |-- Dockerfile
|   |   |-- alembic/
|   |   `-- src/
|   |       |-- api/
|   |       |-- models/
|   |       |-- schemas/
|   |       |-- repositories/
|   |       `-- main.py
|   |
|   |-- mcp-gateway/
|   |   |-- pyproject.toml
|   |   |-- Dockerfile
|   |   `-- src/
|   |       |-- control_plane/
|   |       |   |-- registry.py
|   |       |   |-- policies.py
|   |       |   `-- admin_api.py
|   |       |
|   |       |-- data_plane/
|   |       |   |-- mcp_server.py
|   |       |   |-- router.py
|   |       |   |-- validator.py
|   |       |   `-- audit.py
|   |       |
|   |       |-- adapters/
|   |       |   |-- base.py
|   |       |   `-- http.py
|   |       |
|   |       `-- main.py
|   |
|   `-- agent/
|       |-- pyproject.toml
|       |-- Dockerfile
|       `-- src/
|           |-- ollama_client.py
|           |-- mcp_client.py
|           |-- tool_converter.py
|           |-- loop.py
|           `-- main.py
|
|-- database/
|   |-- migrations/
|   `-- seed/
|
`-- tests/
    |-- unit/
    |-- integration/
    `-- e2e/
```

---

## 22. End-to-end request walkthrough

User enters:

```text
Why did customer_daily_load fail? If the failure is retryable, rerun it.
```

### A. Agent discovers capabilities

```text
Agent -> MCP Gateway: tools/list
Gateway -> Registry DB
Registry -> enabled MCP tools
Gateway -> Agent: MCP tool definitions
```

### B. Agent asks Ollama

The Agent converts those definitions to Ollama tool schemas.

```text
Agent -> Ollama:
user prompt
+
available tool schemas
```

Ollama decides:

```text
get_pipeline(name="customer_daily_load")
```

### C. Agent invokes MCP

```text
Agent -> Gateway:
tools/call
name=get_pipeline
```

### D. Gateway resolves control-plane metadata

```text
Gateway:
tool exists?
enabled?
schema valid?
caller allowed?
which backend route?
```

### E. Gateway invokes REST backend

```text
Gateway -> DataOps API:
GET /pipelines/...
```

DataOps API queries PostgreSQL.

### F. Result returns to Ollama

```text
PostgreSQL
 -> DataOps API
 -> Gateway
 -> Agent
 -> Ollama
```

Ollama may next request:

```text
get_run_status
get_run_logs
get_dq_failures
```

### G. Write operation

If Ollama decides a retry is appropriate:

```text
rerun_failed_run(run_id=...)
```

Gateway checks:

```text
tool enabled
role = operator
risk = write
confirmation present
```

Only then:

```text
POST /runs/{run_id}/rerun
```

### H. Audit

Each call writes:

```text
caller
tool
arguments fingerprint/redacted arguments
backend
status
latency
timestamp
correlation_id
```

This demonstrates the complete gateway architecture in one scenario.

---

## 23. Observability requirements

Every request should receive a correlation ID.

Logs should allow tracing:

```text
Agent request
 -> MCP tool call
 -> gateway decision
 -> backend HTTP request
 -> backend response
 -> MCP result
```

Minimum metrics:

| Metric | Purpose |
|---|---|
| tool call count | Tool usage |
| tool error count | Reliability |
| backend latency | Routing/backend performance |
| denied call count | Policy visibility |
| calls by tool | Usage analysis |
| calls by caller/role | Audit/governance |
| backend HTTP status | Troubleshooting |

Do not log secrets or full sensitive payloads.

For the local lab, structured JSON logs + audit table are sufficient. Prometheus/OpenTelemetry can be added later.

---

## 24. Error model

Normalize backend errors before returning them as MCP errors/results.

Examples:

| Backend condition | Gateway result |
|---|---|
| API 404 | Resource not found |
| API 400 | Invalid tool/backend arguments |
| API 401/403 | Authorization error |
| API 409 | Invalid state / conflict |
| API 5xx | Backend execution error |
| Timeout | Backend timeout |
| Unknown tool | MCP tool not found |
| Disabled tool | MCP tool disabled |
| Schema violation | MCP argument validation error |

The LLM should receive useful but safe errors. Internal stack traces should remain in gateway logs.

---

## 25. Testing strategy

### Unit tests

Focus on:

- registry lookup,
- schema validation,
- policy evaluation,
- path/query/body parameter mapping,
- HTTP adapter,
- error normalization,
- MCP-to-Ollama tool conversion.

### Integration tests

Run against real containers:

```text
Gateway -> DataOps API -> PostgreSQL
```

Validate read and write calls.

### MCP protocol tests

Using the official MCP client:

```text
list_tools
call_tool
```

Verify the Gateway behaves as an actual MCP server.

### Agent tests

Use deterministic prompts and verify that a tool-capable Ollama model can:

```text
discover tools
select tool
provide arguments
consume result
continue multi-turn loop
```

### End-to-end scenario

Seed a failed pipeline execution and validate:

```text
natural-language request
 -> Ollama
 -> MCP
 -> Gateway
 -> REST
 -> PostgreSQL
 -> tool result
 -> Ollama
 -> final explanation
```

---

## 26. Implementation phases

| Phase | Goal | Deliverable |
|---|---|---|
| 0 | Repository bootstrap | Compose, Python projects, CI/test skeleton |
| 1 | Business platform | PostgreSQL + DataOps REST API + OpenAPI |
| 2 | Control plane | MCP registry tables + seed data + admin endpoints |
| 3 | Data plane | Dynamic MCP `tools/list` + `tools/call` |
| 4 | HTTP routing | Generic OpenAPI/HTTP backend adapter |
| 5 | Governance | Role policy, enabled/disabled tools, write confirmation |
| 6 | Agent | MCP client + Ollama client + tool conversion |
| 7 | Agent loop | Multi-turn tool calling and final response |
| 8 | Observability | Audit events + correlation IDs + structured logs |
| 9 | E2E validation | Full failed-pipeline diagnosis/rerun scenario |
| 10 | Hardening | Tests, docs, error cases, clean Compose startup |

Auto-discovery starts only after Phase 10 is working.

---

## 27. Explicit non-goals for this first project

Do not implement these yet:

```text
OpenAPI AutoCrawler
automatic tool generation
vector/semantic tool search
Omni MCP
thousands of tools
gRPC backend adapter
external SaaS MCP servers
production OAuth provider
Kubernetes deployment
multi-region HA
Redis caching
LLM-generated tool descriptions
dynamic credential brokerage
```

The architecture should allow these later, but building them now would hide the core gateway mechanics.

---

## 28. Acceptance criteria

The gateway project is complete when all of the following are demonstrable.

| Capability | Required proof |
|---|---|
| REST backend | 15–20 working OpenAPI endpoints |
| Database | Real PostgreSQL persistence + seeded scenarios |
| Curated MCP catalog | ~10–13 selected tools |
| One MCP endpoint | Agent connects only to Gateway `/mcp` |
| Dynamic tools | `tools/list` comes from registry DB |
| Generic execution | `tools/call` resolves registry route rather than hard-coded tool functions |
| REST translation | MCP calls invoke ordinary backend HTTP endpoints |
| Control plane | Tool can be enabled/disabled without changing the agent |
| Remapping | Tool backend route can change through metadata |
| Policy | Read/write tools have different policy behavior |
| Audit | Every invocation produces audit metadata |
| Ollama | Local model chooses and calls MCP-backed tools |
| Multi-tool flow | Agent solves a scenario requiring 3+ tools |
| Docker | Environment starts from Compose |
| Tests | Unit + integration + one full E2E scenario |

---

## 29. Key design decisions for Codex

Codex should treat the following as architecture constraints, not optional implementation details.

| Decision | Constraint |
|---|---|
| MCP Gateway | Must be generic; no business-specific tool functions in the gateway |
| Tool definitions | Must come from registry/control-plane records |
| Backend protocol | HTTP adapter first, adapter interface preserved |
| Business API | Must remain ordinary REST/OpenAPI, unaware of MCP |
| LLM | Must not be imported into MCP Gateway |
| Agent | Owns Ollama + MCP orchestration |
| OpenAPI | Stable operation IDs required from day one |
| Auto-discovery | Explicitly excluded from initial implementation |
| Tool exposure | Curated subset of REST operations |
| Writes | Must have policy + explicit confirmation behavior |
| Observability | Correlation ID and audit event required |
| Environment | Docker Compose must be the reproducible startup path |

---

## 30. Recommended Codex kickoff statement

Use the following as the initial implementation brief:

> Build the project described in `docs/analysis.md` as an enterprise-style MCP Gateway learning lab. Implement in phases and preserve strict separation between the business REST API, MCP control plane, MCP data plane, Ollama agent, and PostgreSQL. Do not implement OpenAPI auto-discovery yet. The MCP Gateway must dynamically expose curated tools from registry metadata and translate MCP `tools/call` invocations into HTTP requests to the existing DataOps API. Use the official MCP Python SDK over Streamable HTTP. The Ollama integration belongs in a separate Agent service that acts as an MCP client. Every phase must include tests and maintain a runnable Docker Compose environment.

Codex should implement one phase at a time and keep architecture documentation synchronized with code.

---

## 31. Why this is the right pre-AutoCrawler project

This project makes the future AutoCrawler addition obvious.

Today:

```text
OpenAPI
   |
human reviews operation
   |
human creates registry entry
   |
tool becomes available
```

Later:

```text
OpenAPI
   |
AutoCrawler
   |
candidate registry entry
   |
enabled = false
   |
owner review
   |
enabled = true
```

The future crawler changes only **how control-plane records are created**.

It does not change the data-plane architecture.

That is precisely why the Gateway should be built first.

---

## 32. Feasibility and primary risks

### Feasibility

**High.** Every required technical capability exists:

- FastAPI generates OpenAPI automatically.
- The MCP Python SDK supports a remote Streamable HTTP MCP endpoint.
- Its low-level server supports dynamic `tools/list` and `tools/call`, which is important for registry-driven tools.
- Ollama supports function/tool calling and multi-turn agent loops.
- Ollama provides an official Docker image.
- Docker Compose is designed for multi-container local applications.

### Primary implementation risk

The largest risk is not protocol implementation. It is **tool-contract quality**.

If tool names, descriptions, schemas, and errors are poor, a local LLM may select tools incorrectly.

Therefore the first project should keep the catalog to roughly 10–13 carefully designed tools and prove correctness before introducing auto-discovery.

The second risk is local LLM variability. Tool selection quality depends on the model and hardware. The architecture should therefore allow the Ollama model to be changed by configuration.

The third risk is accidentally turning the Gateway into a business-code monolith. Enforce the adapter + registry model: the Gateway should know how to route generic tool calls, not contain ETL-specific business logic.

---

## 33. Official references used for architecture validation

- MCP Python SDK — low-level dynamic server:
  https://py.sdk.modelcontextprotocol.io/v2/advanced/low-level-server/

- MCP Python SDK — Streamable HTTP / ASGI:
  https://py.sdk.modelcontextprotocol.io/run/asgi/

- MCP Python SDK — Client:
  https://py.sdk.modelcontextprotocol.io/client/

- Ollama — Tool calling:
  https://docs.ollama.com/capabilities/tool-calling

- Ollama — Chat API:
  https://docs.ollama.com/api/chat

- Ollama — Docker:
  https://docs.ollama.com/docker

- FastAPI — OpenAPI generation:
  https://fastapi.tiangolo.com/tutorial/first-steps/

- Docker Compose:
  https://docs.docker.com/compose/
