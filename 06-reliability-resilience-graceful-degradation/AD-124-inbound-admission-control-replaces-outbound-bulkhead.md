# AD-124 — Inbound Concurrency-Cap Shedding Is the Live Backpressure AD-50's Removed Bulkhead Was Meant to Provide

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-124 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-13, AD-45, AD-50, AD-123

## Context

AD-50's outbound per-operation bulkhead was designed to bound caller-side concurrency but turned out to be unreachable dead code — removed once it was confirmed the deployed call shape (one A2A call per round per node) left no orchestrator-side fan-out for a semaphore to queue against. That removal closed a mechanism without replacing what it was actually protecting against: Bedrock AgentCore's `InvokeAgentRuntime` enforces a 25 TPS per-agent throttle, and the only defense against exceeding it was reactive — a caller hitting `ThrottlingException`, retrying, and eventually DLQing. A burst of concurrent task requests at a single agent replica had no proactive check on the *receiving* side; it would simply queue or get throttled upstream, with no fast, cheap signal telling the caller to try a different replica sooner.

## Decision

Cap each agent's own in-flight task-execution POST count and shed with HTTP 503 + `Retry-After: 1` once the cap is reached, instead of letting requests queue. Implement as `AdmissionControl`, an ASGI middleware in `buyer_agent_core/admission.py`, wired into every agent's Starlette app via `build_app` (`serve.py`). Guard only HTTP POST (the JSON-RPC task-execution path); GET requests — the agent card and the `/ping` health probe — always pass through, so a saturated agent never appears unhealthy to AgentCore. The cap is a concurrency limit, not a rate: an LLM agent's unit of work is a multi-second two-step run, so the resource to bound is in-flight count, not requests/second. Read once from `A2A_MAX_INFLIGHT` (default 10, deploy-time env var; `<=0` disables it) rather than from governance config, because the agent boot path deliberately reads only model configuration. The `skill_runtime` FastMCP server (port 8000) carries an inlined copy of the same middleware (`MCP_MAX_INFLIGHT`, default 10), since its Docker build context can't import `buyer_agent_core`.

## Alternatives Considered

- **Rely on AgentCore's reactive throttle-then-retry alone.** Rejected: piles more load onto an already-saturated replica while it retries, instead of rescheduling the request against a less-loaded replica sooner.
- **Token-bucket rate limiting (requests/second).** Rejected: doesn't match the resource being protected — a multi-second two-step LLM run is bounded by concurrent count, not arrival rate.
- **Read the cap from `{env}-system-config` like other resilience parameters (AD-45).** Rejected: the agent boot path is deliberately restricted to reading only model configuration; an env var keeps this consistent with other per-agent deploy-time knobs (e.g. `PROMPT_CACHE_ENABLED`) rather than adding a governance read to the hot boot path.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A saturated replica fails fast and cheaply (503 + `Retry-After`) instead of queuing, so the caller's existing retry/backoff (AD-13) reschedules against a less-loaded replica sooner | The cap is an env var, not tunable from `{env}-system-config` — changing it requires a redeploy, unlike the other resilience parameters under AD-45 |
| No lock needed — the single-threaded event loop makes the in-flight counter's read-and-increment atomic between awaits | `skill_runtime`'s MCP server carries its own inlined copy of the middleware rather than sharing the `buyer_agent_core` implementation, a duplication forced by Docker build-context isolation |
| GET requests always bypass the cap, so saturation is structurally invisible to health checks | Live shedding could not be demonstrated end-to-end through the real AgentCore invoke path (see Results) — the defense is offline-proven, not live-proven |

## Results

Shipped and merged to main (PR #45, commit 05d5e34, 2026-06-25); the `skill_runtime` MCP counterpart shipped in the same change. Test coverage: `test_admission.py` (agent path, 3 tests) and `test_mcp_admission.py` (MCP path, 3 tests, drives the real `server.AdmissionControl` so the inlined copy can't drift), plus an httpx `ASGITransport` end-to-end check through a real Starlette middleware stack — 23 tests passing together. Each shed emits the OTEL counter `agent.admission.shed` (or `mcp.admission.shed`) and a warning log.

**Settled negative, 2026-07-02.** A dedicated live test (`test_admission_control_live.py`, now skipped) drove 15–40 concurrent invokes at `dev_kraljic_classifier` and got zero 503s — AgentCore's own invoke front door manages request concurrency such that a burst never stacks past the per-process cap inside one container from the outside. The mechanism is not externally triggerable through the real AgentCore invoke path, and there will not be a live proof of it; it remains valid defense-in-depth (proven offline) rather than a demonstrated live safeguard.

This is the confirmed live replacement for AD-50's removed outbound bulkhead: REQ-R300–R302 (outbound, superseded) give way to REQ-R303/R304 (inbound, live).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
