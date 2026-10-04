# MCPFabric Engineering Instructions

MCPFabric is an enterprise-style MCP Gateway learning project.

## Source of truth

Before making architectural or implementation decisions, read:

- `docs/analysis.md`
- the active implementation plan under `docs/plans/`

Do not contradict architecture decisions in `docs/analysis.md` without
documenting the reason.

## Architecture boundaries

Maintain strict separation between:

1. DataOps REST API
2. MCP Gateway control plane
3. MCP Gateway data plane
4. Agent/orchestrator
5. Ollama
6. PostgreSQL

The business REST API must not know about MCP.

The MCP Gateway must not contain LLM/Ollama logic.

The Agent owns Ollama and MCP client orchestration.

## MCP Gateway

The gateway must be metadata driven.

Do not implement individual business tools as hard-coded MCP functions.

`tools/list` must be generated from registry metadata.

`tools/call` must:

1. resolve tool metadata
2. validate arguments
3. evaluate policy
4. resolve backend route
5. invoke the backend adapter
6. normalize the result
7. write audit information

Start with HTTP as the backend adapter.

Preserve an adapter interface for future protocols.

## OpenAPI

The DataOps API must expose OpenAPI.

Every meaningful REST operation must have a stable `operationId`.

Auto-discovery from OpenAPI is NOT part of the initial implementation.

## Database

Use PostgreSQL.

Keep logical separation between:

- `app` schema
- `mcp` schema

Use migrations.

## LLM

Use Ollama.

The Ollama model must be configurable.

Ollama must not be coupled to the MCP Gateway.

## Runtime

Use Docker Compose as the reproducible local runtime.

The complete project must eventually start with:

`docker compose up --build`

## Quality

For every implementation phase:

- add tests
- run tests
- run lint/type checks if configured
- update relevant documentation
- do not leave knowingly broken code
- do not silently skip acceptance criteria

## Planning

For significant features or architectural work, create/update an ExecPlan
under `docs/plans/` before implementation.

Implement one phase at a time.