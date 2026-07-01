# AD-105 — Warm-Up Sentinel Early Exit (No LLM Cost for Keep-Alive Pings)

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-105 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-059

## Context

`orchestrator/warm_runtimes.py` keeps AgentCore microVMs warm by pinging each agent runtime with a stable `runtimeSessionId` on a <15-minute cadence, resetting the AgentCore idle timer so the next real request avoids a cold start. Each ping carries the payload `{"warmup": true}`.

Before this decision, that payload flowed through the full agent pipeline: `_extract_request` → `prompt_builder` → `Agent(prompt)` → Bedrock model inference. The LLM response was always discarded (the warm-up script tolerates any response, even errors), but the model call itself consumed input tokens — the cached system prefix + tools schema — on every ping. Across 7 agents pinged every ~10 minutes, that is ~30,000 unnecessary LLM calls per month.

## Decision

The A2A executor in `buyer_agent_core.serve._make_executor` short-circuits warm-up pings immediately after request extraction: when `req` is a dict with a truthy `"warmup"` key, the task is marked complete and returns without invoking the runner or the model.

The sentinel payload `{"warmup": true}` is the contract between the warm-up script and every agent entrypoint. The check sits in the shared `buyer_agent_core` package, so all 7 agents inherit it uniformly.

## Alternatives Considered

- **Strip the sentinel in the warm-up script (send an empty payload).** Rejected: an empty string or `{}` would still reach `prompt_builder` and produce a degenerate prompt that wastes an LLM call. The sentinel must be checked inside the agent, after request extraction but before the runner.
- **Check at the prompt-builder level (per-agent).** Rejected: puts the burden on each agent's `prompt_builder` to recognize the sentinel, creating 7 duplicate checks and a risk that a new agent forgets it. The shared executor is the single choke point.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Zero LLM cost for warm-up pings (~30K calls/month eliminated) | A 3-line sentinel check in the hot path (one dict lookup per real request) |
| Single point of enforcement in the shared executor — no per-agent duplication | The `"warmup"` key is reserved; a real request must never include it |

The reserved-key risk is theoretical: no agent's domain schema uses a `"warmup"` field, and the warm-up script controls the only producer of that key.

## Results

Implemented in `packages/buyer_agent_core/buyer_agent_core/serve.py:_make_executor`. The check runs after `_extract_request` and before the runner/LLM path. All 7 agents inherit it through the shared `build_app` → `_make_executor` call chain.

The warm-up script (`orchestrator/warm_runtimes.py`) is unchanged — its `{"warmup": true}` payload now triggers the early exit it always intended.

**Related as-built (PR #89).** The keep-alive only pays off if real traffic lands on the same warm microVM, and AgentCore warmth is per-`runtimeSessionId`: a fresh session ID gets its own cold VM. `warm_runtimes.py` pings with a stable session ID, but three caller paths were minting a new ID per request and so never reused the warmed VM — `skill_client._invoke_skill_tool` (omitted the ID, so boto3 auto-generated a fresh UUID), and the `po_delivery` / `pr_event_router` gateway paths. PR #89 threads stable session IDs through those callers (an `Mcp-Session-Id` header on the gateway paths, whose forwarding to the skill runtime is an untested-but-harmless lever) so the warmth this decision maintains is actually consumed.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
