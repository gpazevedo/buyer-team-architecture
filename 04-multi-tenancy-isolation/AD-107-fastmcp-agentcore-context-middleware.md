# AD-107 — AgentCore Context Middleware for FastMCP

**Theme:** Multi-Tenancy & Isolation  **Catalog:** AD-107 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-6, AD-37, AD-77, AD-99

## Context

AD-6 requires tenant context injection into the execution environment: "tenant context is injected into the execution environment rather than supplied by the model." `BedrockAgentCoreApp` (FastAPI) handles AgentCore Runtime headers (`WorkloadAccessToken`, `OAuth2CallbackUrl`, session ID) automatically — but Buyer Team's 4 MCP servers use FastMCP (`mcp.server.fastmcp.FastMCP`), which produces a bare Starlette app with no AgentCore header processing. Without a bridge, FastMCP servers deployed on AgentCore Runtime have no access to the per-request identity token or tenant context, creating a gap in the context-injection chain that AD-6 requires.

## Decision

A reusable Starlette ASGI middleware (`AgentCoreContextMiddleware`) bridges AgentCore Runtime request headers into `BedrockAgentCoreContext`. It is wired into every FastMCP MCP server at the app level, before any other middleware. When headers are absent (local dev, non-Runtime deployment), the middleware is a no-op — it reads headers, sets them if present, and always passes through. The pattern is adopted from `aws-samples/sample-strands-agent-with-agentcore`.

## Alternatives Considered

- **Switch from FastMCP to BedrockAgentCoreApp for MCP servers.** Rejected: FastMCP auto-generates JSON-RPC tool schemas from Python type hints — switching to BedrockAgentCoreApp would lose this, requiring manual schema maintenance and breaking the existing MCP client contract.
- **Manual header extraction per tool.** Rejected: duplicates boilerplate across every tool function; error-prone when adding new tools or servers.
- **Status quo (no injection).** Rejected: FastMCP servers on AgentCore Runtime would operate without tenant identity, violating AD-6 and preventing ABAC credential derivation (AD-37).

## Trade-offs

| Gained | Given up |
| --- | --- |
| FastMCP servers on AgentCore Runtime now have the same header injection as BedrockAgentCoreApp — AD-6 context injection is complete | A dependency on the `bedrock-agentcore` PyPI package (v1.15.1) — previously only available at runtime; now a build-time dependency for all MCP servers |
| Single reusable middleware across all 4 MCP servers; no per-tool changes needed | Middleware is wired in all servers even though only `skill_runtime` currently runs on AgentCore Runtime — the others are local-dev only; the no-op-when-headers-absent design makes this safe but carries unused code |
| Pattern is identical to the AWS sample repo's middleware — externally validated approach | Starlette `BaseHTTPMiddleware` adds one async stack frame per request; negligible for MCP tool latency but real |

## Results

Wired into all 4 MCP servers: `skill_runtime` (port 8000, the primary Runtime-deployed server), `dynamodb_master_data`, `step_functions_orchestrator`, and `tenant_mdm_emulator`. The middleware sits on the innermost app (before `AdmissionControl` wraps it) so headers are extracted regardless of admission state. When AgentCore Runtime sends `WorkloadAccessToken` on an invocation, `BedrockAgentCoreContext.get_workload_access_token()` is populated inside the tool handler — enabling downstream JWT claim extraction, tenant scoping, and the ABAC credential path (AD-37). Verified with 6 unit tests (header extraction, session/request IDs, all-headers, no-header noop, pass-through) + 163 existing tests (zero regressions).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
