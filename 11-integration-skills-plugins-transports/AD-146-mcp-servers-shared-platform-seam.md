# AD-146 — `mcp_servers/shared` Becomes the MCP Servers' Platform Seam

**Theme:** Integration, Skills, Plugins & Transports
**Catalog:** AD-146 · **Source PRD:** PRD-013 · **Status:** Accepted · **Related:** AD-107, AD-135, AD-145, AD-116, AD-099

## Context

AD-107 wired `AgentCoreContextMiddleware` into all 4 FastMCP servers as a single reusable piece — tenant-context injection specifically. Cross-cutting concerns beyond tenant context were not similarly unified: each server built its Starlette app directly, and full adoption of even the existing shared middleware was never independently verified. `skill_runtime` — the one server actually deployed on AgentCore Runtime, and therefore the one under real production load — still imported `AgentCoreContextMiddleware` directly rather than through a shared factory, ran synchronous `boto3` (DynamoDB, STS, Step Functions) directly inside async tool handlers on FastMCP's single-worker `uvicorn` process with no explicit client timeout, and had never wired the `/ping` route the other three servers already exposed through a shared app-construction path. A stalled AWS call — throttling, a network blip — froze that one process's event loop for every other tenant's in-flight request, well past what its callers wait (Node 7 Lambda's 120s, the Gateway's 50s), and there was no way to detect the stalled state from outside, since the server had no readiness route to report it on.

## Decision

Consolidate `mcp_servers/shared`'s cross-cutting code around one factory, `build_mcp_app()` (new `app.py`), replacing the standalone `otel_instrumentation.py` module with `observability.py` and adding `health.py` (the `/ping` route) alongside the existing `errors.py` and a new `tenant.py`. `build_mcp_app` wires AD-107's context middleware, observability instrumentation, and the health route in one call, so a server built through the factory gets all three by construction instead of each being separately remembered per server. `dynamodb_master_data`, `step_functions_orchestrator`, and `tenant_mdm_emulator` adopt the factory directly.

`skill_runtime` is switched from its direct `AgentCoreContextMiddleware` import to `build_mcp_app` in the same change that fixes its blocking-call bug: the synchronous calls inside async tool handlers (`_s3_prefix`, `_get_master_pr`, `_claim_master_pr`, `_upsert_domain_requisition`, `_start_workflow`) are offloaded via `asyncio.to_thread`, and its DynamoDB/STS/Step Functions clients gain the same tight `Config` (10s connect / 30s read / adaptive retries) that `mcp_clients.py` — the client-side half of this same server — already used, closing an asymmetry where the server's own outbound AWS calls had no timeout while its client-side calls did.

## Alternatives Considered

- **Fix `skill_runtime`'s blocking calls and missing `/ping` in isolation, leave `mcp_servers/shared`'s structure as-is.** Rejected: the fix already needed a shared app-construction entry point to pick up the health route without hand-wiring it a fourth way; building that entry point as the shared factory costs the same effort and stops the next server from repeating the same two gaps.
- **Give `skill_runtime` its own dedicated timeout/threading wrapper instead of adopting the shared factory the other servers use.** Rejected: three of the four servers already went through some form of shared construction; special-casing the one server with the most production exposure makes it the exception rather than the reference implementation.
- **Move to a fully async `boto3` client (`aioboto3` or similar) instead of `asyncio.to_thread`.** Rejected for now: it would touch every AWS call site's construction and error-handling shape across all four servers, where `asyncio.to_thread` is a call-site-local wrapper around the same synchronous `boto3` calls the other servers and `mcp_clients.py` already rely on — a smaller surface for the same event-loop-safety property.

## Trade-offs

| Gained | Given up |
| --- | --- |
| `skill_runtime` — the only AgentCore-Runtime-deployed server — no longer blocks its single event loop on a stalled AWS call; one tenant's slow request can no longer freeze every other tenant's in-flight request on the same process | Every offloaded call now round-trips through a thread-pool hop, a small latency cost on the common (non-stalled) path in exchange for the tail-latency protection |
| A server built via `build_mcp_app` gets tenant-context middleware, observability, and `/ping` by construction — the next new MCP server can't forget any of the three the way `skill_runtime` forgot `/ping` | `otel_instrumentation.py`'s removal means any code that imported it directly, rather than through the shared observability wiring, needs to move to `observability.py` — a one-time migration cost across the 4 servers |
| `skill_runtime`'s outbound AWS calls now carry the same timeout discipline its own client side already had — closes an asymmetry that existed only because the two sides of the same server were written at different times | The bounded timeout (10s connect / 30s read) is a policy choice carried over from `mcp_clients.py` without independently re-deriving it for the server side's different call shapes |

## Results

Shipped across two impl PRs merged 2026-08-16: #322 (`feat(mcp_servers): shared cross-cutting base for AgentCore MCP servers`) and #323 (`fix(skill_runtime): stop blocking the event loop on AWS calls, adopt shared build_mcp_app`). All four MCP servers now construct their app through the same entry point; `skill_runtime` gained the `/ping` route the other three already had. AD-107 stays accurate on the middleware itself and is unchanged — this decision wraps it inside a broader factory rather than superseding it. This is the third instance of the "one platform seam per layer" pattern, after `buyer_agent_core` (AD-135, the agent layer) and `lambda_core` (AD-145, the Lambda layer) — each closes the same shape of drift (independently constructed clients, copy-pasted cross-cutting code, a convention nobody enforced) in its own runtime, one layer at a time.

**Update 2026-08-16 (impl PR #324): the seam gains its structural check, on a narrower basis.** `mcp_servers/tests/test_mcp_server_boundary.py` extends the same AST-walk enforcement to the MCP layer — but where `lambdas/tests/test_lambda_layer_boundary.py` (AD-145) enforces a shared-imports-only rule, this server seam has no top-level `__all__` (servers import submodules directly) and no `aws.py` yet, so its test is narrower: an allow-list plus a no-dynamic-import check only, with `boto3` approved and documented as a known gap rather than papered over. It is the same shape of guarantee AD-135/AD-145 ship for the agent and Lambda layers, scaled to what this layer's structure currently makes enforceable.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
