# AD-123 — Readiness Gate + SIGTERM Draining Close the Fail-Open Window at Both Ends of a Replica's Life

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-123 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-45, AD-46, AD-50, AD-124

## Context

Each of the 7 LLM agents serves its A2A endpoint from a long-lived Starlette/uvicorn process inside a single-session AgentCore microVM — there is no fleet of interchangeable instances to fail over to (AD-50's context). Two independent gaps existed at the edges of that process's lifecycle, both with the same shape: traffic reaching a replica that is not actually able to serve it. At startup, `/ping` was a bare liveness probe — it reported Healthy the instant the process bound its port, before the blueprint had resolved a model or completed prompt warm-up; a cold or half-initialized replica could take real task traffic and fail it immediately. At shutdown, AgentCore sends SIGTERM on scale-in or redeploy; a bare `uvicorn.run()` either dropped in-flight requests immediately or, if allowed to finish, kept reporting Healthy on `/ping` the whole time it drained, so new traffic kept routing to a process already on its way out.

## Decision

Turn `/ping` into a real readiness-and-drain gate, not a liveness check. Track state in a small `_Health` object (`ready`, `draining`) stored on `app.state`; `/ping` reports Healthy only when `ready and not draining`. `ready` flips `True` from a Starlette lifespan startup hook only after `_blueprint_ready()` confirms the blueprint resolved a model and assembled its system prompt, and after an optional best-effort `warmup` callable has run (a warm-up failure logs but never blocks readiness). On SIGTERM/SIGINT, `_install_drain` wraps uvicorn's own `handle_exit` to flip `draining = True` — so `/ping` goes Unhealthy — *before* delegating to uvicorn's native graceful shutdown, which then waits up to `A2A_GRACEFUL_TIMEOUT` (default 30s) for the in-flight `asyncio.to_thread` two-step run to finish before the process exits. All 7 agent entrypoints share this through `buyer_agent_core/serve.py`'s `build_app` + `serve`, rather than each calling `uvicorn.run()` directly.

## Alternatives Considered

- **Bare liveness check only (200 once the port is bound).** Rejected: this is the exact gap that caused the failure mode — a replica can be listening long before it can actually serve a task.
- **Immediate hard shutdown on SIGTERM, no drain.** Rejected: drops in-flight LLM runs mid-execution — the same "process dies before anything can react" failure class the timeout-ordering fix (AD-46) exists to prevent on the calling side. Draining is the shutdown-side half of that same discipline.
- **A separate sidecar/external health-check process.** Rejected: adds an operational component to keep in sync with in-process readiness state for no benefit — AgentCore already polls `/ping` directly.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Cold or half-initialized replicas never take task traffic — no first-call failures against a warming process | A broken blueprint now surfaces as a stuck-Unhealthy replica (logged, not crashed) rather than an immediate failure — requires watching logs to diagnose |
| Draining replicas stop receiving new traffic before their socket closes, so scale-in/redeploy never truncates an in-flight run | `A2A_GRACEFUL_TIMEOUT` (30s default) is an assumption, not a confirmed value against AgentCore's actual SIGKILL grace period — too short and an in-flight run is killed anyway |

The two halves are deliberately one mechanism (`_Health.status`), not two independent flags, so there is exactly one place — ready-and-not-draining — that decides whether a replica is allowed to take traffic.

## Results

Implemented in `packages/buyer_agent_core/buyer_agent_core/serve.py` (`_Health`, `_blueprint_ready`, `build_app`, `_install_drain`, `serve`), used by all 7 agent entrypoints. Landed in two stages: the readiness gate first (commit a731d2e), then SIGTERM draining (PR #52, commit 3c73bf9, merged 2026-06-26), which replaced the readiness gate's original bare `ready=[False]` flag with the combined `_Health` object. A real-SIGTERM integration test (PR #56, merged 2026-06-27, `test_drain_integration.py`) spawns the actual `serve()` runner as a subprocess, fires a slow in-flight request, sends a genuine OS SIGTERM mid-flight, and asserts the request still completes while `/ping` reports draining — proving the mechanism against real uvicorn rather than mocks. `A2A_GRACEFUL_TIMEOUT`'s correctness against AgentCore's real termination grace period remains unconfirmed and is the main open risk. Complements AD-124's inbound admission control: admission control sheds excess *new* requests while a replica is healthy; readiness/draining governs whether the replica should be receiving requests *at all*.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
