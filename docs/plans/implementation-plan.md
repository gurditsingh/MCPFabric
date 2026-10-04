# MCPFabric Implementation Plan

## Status and source of truth

Architecture validation and planning only; no application implementation is authorized by this task. All phases below are pending. Implement one phase at a time and update this plan with results, decisions, and deviations.

Read completely: `AGENTS.md` and `docs/mcp_gateway_analysis.md` (sections 1–33). The requested `docs/analysis.md` does not exist. This plan uses the existing specification without duplicating or renaming it. Resolve that documentation path mismatch during bootstrap by choosing one canonical file and updating references together.

## Target architecture and validation

The Agent discovers a curated MCP catalog, asks a local model which tools to use, executes those calls through one gateway, and returns an explanation. The gateway translates protocol requests into ordinary REST calls using database metadata. The business API manages DataOps state without knowing that its callers use MCP. PostgreSQL persists both business data and gateway governance data in separate schemas.

| Component | Responsibility | Boundary |
|---|---|---|
| PostgreSQL | Durable `app` and `mcp` schemas, migrations, realistic seed scenarios | One local container; separate service ownership and credentials |
| DataOps REST API | Pipelines, runs, logs, DQ failures, datasets, lineage, incidents; FastAPI OpenAPI | Owns `app`; no MCP or Ollama dependencies |
| Gateway control plane | Curated tool schemas, service registrations, routes, mappings, policies, admin endpoints | Owns `mcp`; admin operations are never model tools |
| Gateway data plane | Streamable HTTP `/mcp`, dynamic listing, validation, authorization, generic routing, result normalization, audit | Reads registry; calls adapters; no business-specific handlers or LLM imports |
| Agent/orchestrator | MCP client, schema conversion, Ollama client, bounded conversation loop, user confirmation | Knows gateway and Ollama addresses, never backend URLs or DB credentials |
| Ollama | Configurable local tool-capable model and inference API | Separate container; consumed only by Agent |

Control and data planes are separate modules within one gateway process initially, not separate deployments. Admin REST routes and MCP transport share port 8000. API uses 8080, Ollama 11434. These are planned runtime contracts, not existing endpoints.

The proposed approach is feasible: the official SDK documents [low-level handlers with explicit schemas and Streamable HTTP](https://py.sdk.modelcontextprotocol.io/v2/advanced/low-level-server/) and [ASGI integration](https://py.sdk.modelcontextprotocol.io/run/asgi/); Ollama documents [multi-turn tool calling](https://docs.ollama.com/capabilities/tool-calling). Pin and test a compatible SDK release during bootstrap; do not mix SDK major-version examples. Low-level handlers require application-owned argument validation and result construction.

### Mandatory execution contracts

- `tools/list` queries enabled registry records, never a static Python tool list. Begin without a registry cache; subsequent calls observe committed metadata changes.
- `tools/call` resolves a consistent tool/route/policy snapshot, checks enabled status, validates arguments, evaluates policy, resolves the adapter, invokes it, validates/normalizes output, and records an outcome. Unknown, disabled, invalid, denied, failed, and successful calls all require audit records.
- `BackendAdapter.invoke(route, arguments, context) -> GatewayResult` is protocol-neutral. Implement only HTTP initially. Store path/query/body/header mappings explicitly; no LLM inference, business-name dispatch, or per-tool decorators.
- Preserve manually assigned, unique OpenAPI `operationId` values. Maintain a checked-in method/path/operation-ID/MCP mapping manifest. Changing Python function names must not change these IDs. OpenAPI validates contracts; it does not create registry records.
- Read/write classification and allowed roles live in metadata. Rerun/cancel require operator role plus explicit user confirmation. Incident creation is a write with role checks and audit, but no confirmation by default, matching the specification's policy table.
- Confirmation belongs to Agent-controlled execution context, not model-provided arguments. Bind approval to caller, tool, and canonical argument fingerprint; reject mismatches and reuse. Validate the SDK request-metadata mechanism before selecting its concrete wire representation. Never pass confirmation or identity fields into business request bodies.
- Local `X-User`/`X-Role` identity is a lab assumption, not authenticated production identity. Protect admin endpoints with a separate local admin credential; model callers cannot edit their policies.
- No automatic retries for side-effecting backend requests. A timeout can mean an unknown outcome; surface that explicitly rather than duplicating a write.

## Dependency and runtime order

Implementation follows `bootstrap -> app persistence/API -> registry/admin -> MCP runtime -> HTTP routing -> governance -> Agent clients -> Agent loop -> observability -> E2E -> hardening`.

Runtime dependencies are:

```text
PostgreSQL healthy -> migration job succeeds -> seed job succeeds
  -> DataOps API ready
  -> Gateway ready (DB required; API checked before backend invocation)
Ollama ready -> model-init succeeds -> configured model available
Gateway ready + Ollama model available -> Agent ready
Agent -> Gateway -> DataOps API -> PostgreSQL
Agent -> Ollama
```

Migration and seed jobs complete before dependent services start; health checks alone do not prove schemas exist. Ollama can start independently. Neither gateway startup nor backend tests require a model. CPU execution is the baseline; GPU is optional. Model acquisition may require an initial network download and persistent model volume.

## Proposed final repository structure

```text
MCPFabric/
  AGENTS.md
  README.md
  compose.yaml
  .env.example
  .gitignore
  Makefile
  pyproject.toml                   # uv workspace and shared test/check configuration
  uv.lock
  .github/workflows/ci.yml
  docs/
    analysis.md                   # canonical name after reference reconciliation
    architecture.md
    api-tool-mapping.md
    plans/implementation-plan.md
  services/
    dataops-api/
      pyproject.toml
      Dockerfile
      src/dataops_api/
        main.py
        config.py
        api/{pipelines,runs,datasets,incidents,health}.py
        models/
        schemas/
        repositories/
        execution.py
    mcp-gateway/
      pyproject.toml
      Dockerfile
      src/mcp_gateway/
        main.py
        config.py
        context.py
        models/
        control_plane/{registry,policies,admin_api,schemas}.py
        data_plane/{mcp_server,router,validator,results,audit}.py
        adapters/{base,http,mapping}.py
    agent/
      pyproject.toml
      Dockerfile
      src/mcpfabric_agent/
        main.py
        config.py
        ollama_client.py
        mcp_client.py
        tool_converter.py
        confirmation.py
        loop.py
  database/
    alembic.ini
    migrations/{env.py,versions/}
    seed/{app.json,mcp.json,loader.py}
    contracts/api-tool-map.json
  tests/
    conftest.py
    fixtures/
    unit/{dataops,gateway,agent}/
    integration/{api,registry,mcp,routing,policy,audit}/
    e2e/
```

Brace notation describes proposed files/directories, not literal names. Use one migration history under `database/` to avoid competing schema owners; migrations explicitly identify `app` or `mcp` ownership. Tool seed JSON is curated configuration, not executable gateway business logic. Optional tool-definition history can be added later.

## Common phase gate

Every phase adds and runs relevant pytest/pytest-asyncio tests, configured lint/type checks, and documentation updates. Record exact commands and results here. Proposed root commands are `uv sync --frozen`, `uv run pytest tests/unit`, `uv run ruff check .`, and `uv run mypy services`; bootstrap must make them executable and document integration/E2E prerequisites. These commands are not available today.

Use real PostgreSQL for persistence integration tests, an official MCP client for protocol tests, and controlled HTTP fixtures for adapter failure tests. Mock Ollama only for deterministic Agent unit tests; live-model acceptance is separate and mandatory. No invented coverage percentage replaces behavioral acceptance. Each phase preserves a runnable partial Compose stack; unavailable future services are added only when implemented.

### Phase 0 — Bootstrap and contract decisions

- **Objective:** Establish reproducible packaging, checks, and runtime foundations.
- **Files/modules:** Root manifests/lockfile, Makefile, CI, `.env.example`, Compose, service manifests/Dockerfiles, test fixtures, architecture and mapping docs.
- **Work:** Choose compatible dependency versions; establish importable package layout, CPU baseline, non-secret configuration, DB health checks, and migration job conventions. Reconcile analysis filename. Record identity, confirmation transport, and error-envelope contracts before runtime handlers.
- **Tests:** Package import/build smoke checks, configuration validation, dependency-boundary checks rejecting Ollama in gateway and MCP in business API.
- **Acceptance:** Locked dependencies, documented checks, valid Compose configuration, no business endpoints/tools introduced in bootstrap.
- **Docker/runtime:** `docker compose config`; start PostgreSQL and verify readiness; build available skeleton images. Probe SDK client/server transport in an isolated test, not an implemented business service.

### Phase 1 — PostgreSQL and DataOps API

- **Objective:** Provide real persisted business operations with stable OpenAPI contracts.
- **Files/modules:** `dataops_api` API/models/schemas/repositories/execution modules; `app` migrations and seeds; mapping manifest; API tests.
- **Work:** Implement all 19 operations from specification section 6, including health, `/docs`, and `/openapi.json`. Seed successful, DQ-failed, retryable-infrastructure-failed, running executions, lineage, and open/resolved incidents. Use deterministic lab execution transitions rather than a real ETL engine. Support name filtering in `list_pipelines`; Agent resolves names to IDs before `get_pipeline` or run calls. Include latest run references in pipeline responses.
- **Tests:** CRUD, relationships, state conflicts, rerun/cancel, incident persistence, seed repeatability; compare all operation IDs against manifest and assert uniqueness.
- **Acceptance:** 19 documented working operations, persistent state across restart, predictable success and repeated-failure rerun fixtures; no MCP dependencies in API.
- **Docker/runtime:** Start DB/migrations/seeds/API; fetch OpenAPI, invoke reads/writes, restart API/DB without deleting volumes, and verify persisted records.

### Phase 2 — Registry and admin control plane

- **Objective:** Persist and manage the curated catalog independently of Agent code.
- **Files/modules:** Gateway models, registry/admin schemas/admin API, `mcp` migrations/seeds, admin configuration and registry tests.
- **Work:** Create servers/tools/routes/policies/audit tables. Seed exactly the 13 tools in section 7 with descriptions, input/output schemas, owner, risk, enabled flag, operation IDs, and explicit mappings. Implement all seven section-10 admin routes; audit listing may initially return an empty dataset. Validate schema/mapping references and protect admin writes. Use transactions for related metadata updates.
- **Tests:** Registry CRUD, role/credential rejection, invalid schemas/routes, duplicate names, persistence, enable/disable, remapping, idempotent seeds that do not overwrite operator edits.
- **Acceptance:** Catalog excludes pipeline deletion and admin tools; edits persist with valid references; no backend execution yet.
- **Docker/runtime:** Run gateway admin API against PostgreSQL; modify and restart, verify metadata survives; no Ollama dependency.

### Phase 3 — Dynamic MCP data plane

- **Objective:** Establish genuine Streamable HTTP MCP with generic dispatch.
- **Files/modules:** MCP server/router/validator/results/context/audit modules; adapter interface; protocol tests.
- **Work:** Mount `/mcp`, implement dynamic listing and registry-based call resolution. Validate input JSON Schema and enabled state; establish policy gate and audit lifecycle now. Use a test adapter to exercise generic dispatch; unimplemented production adapters deny execution explicitly. No temporary allow-all write path.
- **Tests:** Official client initialization/list/call, multiple schemas, unknown/disabled/invalid calls, independent catalog edits, denied writes, audit outcome records. A test-only arbitrary tool proves no tool-name dispatch branch is needed.
- **Acceptance:** Database changes alter next listing/call without rebuild; protocol-compliant results; model-independent runtime; no live business writes before governance.
- **Docker/runtime:** Official MCP client connects to container `/mcp`; listing works with Ollama absent; stub adapter exists only in tests.

### Phase 4 — Generic HTTP execution

- **Objective:** Translate tool calls into real REST requests deterministically.
- **Files/modules:** HTTP adapter, mapping, result normalization, routing tests, API-tool manifest validation.
- **Work:** Map explicit path/query/body/header fields, safely encode values, use configured service origins and httpx timeouts. Prevent arbitrary caller-supplied URLs/credentials. Validate output schemas. Normalize 400, 401/403, 404, 409, 5xx, timeout, and malformed responses without exposing stack traces.
- **Tests:** Mapping/encoding and response validation; real gateway/API/PostgreSQL reads; controlled backend failure cases; change metadata route to an equivalent fixture endpoint and prove invocation changes without gateway/Agent edits.
- **Acceptance:** Router uses only adapter contract; all read tools work; writes remain denied until phase 5. Remapping is demonstrable through metadata alone.
- **Docker/runtime:** Exercise MCP reads against real API and DB; stop API and verify bounded safe failures and audit records.

### Phase 5 — Governance and confirmed writes

- **Objective:** Enforce metadata-defined role and confirmation policies before side effects.
- **Files/modules:** Policy interface/evaluator, caller context, confirmation validation, admin tests, policy integration tests.
- **Work:** Enforce section-14 roles; default-deny missing identity/policy. Implement bound confirmation context and consume approvals for rerun/cancel. Preserve enabled-state checks on every call; do not infer permission from tool annotations. Keep production authentication replaceable.
- **Tests:** Developer/operator/admin matrix; missing/false/mismatched/replayed confirmation; denied requests never reach backend; approved rerun/cancel and role-authorized incident creation persist and are audited.
- **Acceptance:** Read/write behavior differs as specified; metadata updates change decisions; disabled tools cannot execute even from stale Agent catalogs.
- **Docker/runtime:** Official MCP client supplies local caller context; verify denied and approved writes against real containers and resulting app/audit rows.

### Phase 6 — Agent clients and configurable Ollama

- **Objective:** Integrate model inference and MCP exclusively in Agent.
- **Files/modules:** Agent configuration, MCP/Ollama clients, tool converter, Agent Dockerfile, Ollama/model-init Compose services, client tests.
- **Work:** Discover schemas from gateway and convert supported shapes faithfully to Ollama tools; explicitly reject unsupported schema features. Configure `OLLAMA_MODEL`, base URLs, timeouts, and caller context. Document model download/readiness and CLI access.
- **Tests:** Conversion contracts, connection failures, argument parsing, no backend URL/DB configuration in Agent; gateway dependency checks continue to exclude Ollama.
- **Acceptance:** Live configurable model selects a valid discovered read tool; MCP client executes it; changing model configuration needs no gateway changes.
- **Docker/runtime:** Start Ollama/model-init/Agent on CPU; verify selected model exists, direct inference works, and Agent connects only to gateway `/mcp` for tools.

### Phase 7 — Bounded Agent loop and user confirmation

- **Objective:** Complete multi-turn discovery, execution, and final explanation.
- **Files/modules:** Agent loop/confirmation/CLI, deterministic model fixtures, loop tests, usage docs.
- **Work:** Append tool results with matching call identifiers; handle multiple calls and errors; enforce turn/call/time limits. Refresh catalog per task and on stale-tool failure. Execute writes serially; prompt the user with exact tool/arguments before approval. Noninteractive mode requires explicit pre-authorized tool/argument approvals, never model-generated confirmation.
- **Tests:** Mocked multi-turn sequences, invalid arguments, disabled tools, loop limits, user refusal, repeated write calls, model output with fabricated confirmation, final answer using returned evidence.
- **Acceptance:** Agent performs 3+ distinct tools and terminates with grounded explanation; lack of approval blocks rerun/cancel. No backend logic migrates into Agent.
- **Docker/runtime:** Run interactive CLI using `docker compose exec agent ...` (exact entrypoint documented when implemented); exercise one read workflow and one human-approved rerun with live Ollama.

### Phase 8 — Observability and audit hardening

- **Objective:** Trace every tool invocation end to end and safely inspect outcomes.
- **Files/modules:** Audit/logging/context modules, API correlation middleware, admin audit query, observability tests/docs.
- **Work:** Complete early audit plumbing with caller, tool, redacted arguments/fingerprint, backend, status, latency, timestamp, correlation ID. Include denied/unknown calls. Propagate correlation across Agent/gateway/API. Emit structured JSON logs and audit-derived counts for calls/errors/denials, tool/caller/role breakdowns, latency, and HTTP status. Reserve durable audit attempt before writes; refuse execution if reservation fails; distinguish unfinished attempts from confirmed outcomes.
- **Tests:** Every outcome yields audit data; redaction tests; correlation propagation; audit outage prevents write dispatch; backend success followed by audit-finalization failure retains an incomplete attempt without claiming rollback.
- **Acceptance:** Admin audit search and logs reconstruct a workflow; sensitive payloads and credentials are absent. Database failure is reported, never silently converted into an unaudited success.
- **Docker/runtime:** Run reads, writes, denials, and backend failure; inspect correlated JSON logs and PostgreSQL rows through `/admin/audit`.

### Phase 9 — Full E2E acceptance

- **Objective:** Prove the learning scenario through the entire live stack.
- **Files/modules:** E2E tests, repeatable seed scenarios, runtime test instructions, evidence fixtures.
- **Work:** Diagnose `customer_daily_load`; resolve pipeline/run IDs through tools; inspect status/logs/DQ; retry only infrastructure failures with operator approval; create incident if retry fails. Provide deterministic API outcomes, fixed prompts, and documented model tag/settings; record model variability separately from deterministic protocol tests.
- **Tests:** Live Ollama natural-language flow through MCP/HTTP/PostgreSQL with 3+ distinct tools and grounded explanation; successful retry, failed retry plus incident, DQ failure without retry, unauthorized and unconfirmed writes. Inspect database side effects and audit trace, not only prose.
- **Acceptance:** End-to-end chain and safety branches pass; metadata disable/remapping remains effective with an unchanged Agent.
- **Docker/runtime:** `docker compose up --build`; run E2E suite against real containers/model. Report model/hardware/time and failures; never substitute mocks for this gate.

### Phase 10 — Reproducibility and hardening

- **Objective:** Close acceptance gaps and document a reproducible learning lab.
- **Files/modules:** CI, Compose health/readiness, configuration validation, full regression tests, README/architecture/mapping documentation.
- **Work:** Verify clean-volume migration/seed startup and existing-volume restart; pin images/dependencies/model tag where possible. Confirm resource/time limits, safe error handling, admin separation, DB privileges, and import boundaries. Document first-run network/model requirements and all local URLs. Re-run acceptance matrix.
- **Tests:** Unit, real-container integration, official MCP protocol, live-model E2E, lint/types; stable operation-ID regression and new metadata-only tool regression.
- **Acceptance:** All section-28 proofs below have recorded passing evidence; no knowingly broken phase or silently skipped gate.
- **Docker/runtime:** Clean isolated Compose project starts with `docker compose up --build`; restart preserves data and metadata; gateway/API remain usable while Ollama is stopped. Do not delete contributor volumes for tests.

## Risks, assumptions, and unresolved decisions

| Risk/assumption | Resolution or gate |
|---|---|
| Analysis filename differs from instructions | Bootstrap reconciles canonical path and all references; no speculative duplicate source |
| SDK versions differ across referenced examples | Phase 0 pins one release and proves transport, lifecycle, validation, metadata access |
| Tool schemas/descriptions affect model choices | Curate 13 tools; contract tests plus live-model E2E; reject unsupported conversion explicitly |
| Model availability, CPU latency, and nondeterminism | Configurable model, persistent volume, bounded loop; record live evidence and resource needs in phases 6/9 |
| Pipeline-name walkthrough conflicts with ID-only API | Phase 1 name-filtered listing returns IDs/latest run; Agent resolves names rather than changing gateway business logic |
| Real ETL execution unspecified | Deterministic lab execution simulator in API only; fixtures cover both retry outcomes |
| Confirmation wire format unspecified | Phase 0 transport probe selects application metadata representation; phase 5 validates binding/reuse; model cannot authorize itself |
| Simulated headers can be forged | Local lab only; separate admin credential and replaceable policy/context interfaces; production identity deferred |
| Metadata edits race with calls | Per-call consistent snapshot; committed edits apply to subsequent calls; no mid-flight cancellation promise |
| Database audit and HTTP side effects are not atomic | Durable attempt before dispatch, incomplete-outcome tracking, no blind write retries; phase 8 failure tests |
| Registry becomes a hard-coded tool collection | Arbitrary metadata-only tool and route-remap tests; no per-tool handler/decorator/name switch |
| Routing can leak secrets or reach arbitrary hosts | Registered origins, explicit mappings, filtered headers, timeouts, redaction; tests in phases 4/8/10 |
| Shared PostgreSQL blurs boundaries | `app`/`mcp` ownership and least-privilege service credentials; single migration history |

Open decisions are implementation details to settle at their phase gates, not missing acceptance requirements. Do not claim production-grade authentication, distributed atomicity, or deterministic LLM behavior.

## Explicitly out of scope

OpenAPI auto-discovery/AutoCrawler, automatic registry generation, LLM-generated tool descriptions, semantic/vector tool search, Omni MCP, thousands of tools, gRPC/native-MCP/Lambda adapters, external SaaS MCP servers, production OAuth, dynamic credential brokerage, Kubernetes, multi-region HA, Redis caching, and a large admin UI. Real ETL execution infrastructure is outside this lab; the API owns its simulation.

AutoCrawler may be planned only after phase 10 is accepted. Its future output would be disabled candidate registry records keyed to stable operation IDs and subject to human review; it must not change the generic runtime execution contract.

## Acceptance review against specification section 28

All criteria are covered by planned work; none is implemented or verified yet.

| Capability | Planned proof | Phase | Planning gap |
|---|---|---|---|
| REST backend: 15–20 OpenAPI endpoints | 19 section-6 operations exercised; unique stable ID manifest | 1, 10 | None |
| Database: real persistence and seeds | Both PostgreSQL schemas, realistic scenarios, restart persistence | 1, 2, 10 | None |
| Curated catalog: ~10–13 tools | Exact 13 section-7 records; deletion/admin excluded | 2, 3 | None |
| One MCP endpoint | Agent configuration/traffic uses gateway `/mcp` only | 3, 6, 9 | None |
| Dynamic tools | Registry edit changes next `list_tools` response without rebuild | 2, 3 | None |
| Generic execution | Arbitrary metadata-only tool routes through adapter without Python edits | 3, 4, 10 | None |
| REST translation | Official MCP calls reach ordinary API and persist/read business state | 4, 5 | None |
| Control plane | Enable/disable via admin observed by unchanged Agent; stale calls denied | 2, 3, 7, 9 | None |
| Remapping | Metadata route update redirects execution to equivalent fixture backend | 4, 9 | None |
| Policy | Role matrix, rerun/cancel approval, incident role checks, no denied side effects | 5, 7 | None |
| Audit | Every invocation outcome traced, including unknown/denied/error calls | 3, 5, 8 | None; outage semantics explicitly tracked |
| Ollama | Live local configurable model selects and executes discovered tools | 6, 7, 9 | None; model compatibility must be measured |
| Multi-tool flow | Failed-pipeline scenario uses 3+ distinct tools and grounded response | 7, 9 | None |
| Docker | Clean and existing-volume startup with prescribed Compose command | 0–10 | None; first-run model download documented |
| Tests | Unit, real-container integration, protocol, and full live-model E2E evidence | All; final gate 10 | None |

Section-29 constraints are also preserved: generic registry-driven gateway; HTTP adapter abstraction; MCP-unaware REST API; Agent-only Ollama; stable operation IDs; curated exposure; policy/confirmation; correlation/audit; Compose runtime; no auto-discovery.

### Review result and progress

- [x] Read available contributor instructions and full architecture specification.
- [x] Validate boundaries, dependencies, phases, tests, runtime gates, risks, and final layout.
- [x] Map all 15 acceptance criteria to explicit proofs; no omitted planning criteria.
- [ ] Reconcile missing `docs/analysis.md` path during bootstrap.
- [ ] Resolve SDK/confirmation transport details and model compatibility at designated gates.
- [ ] Implement and execute phases 0–10 in future authorized work.

The remaining gaps are the source-path mismatch and unvalidated runtime choices, plus all implementation evidence. This documentation does not assert that any service or acceptance test currently passes.
