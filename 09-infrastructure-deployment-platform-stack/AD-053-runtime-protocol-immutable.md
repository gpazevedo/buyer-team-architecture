# AD-053 — AgentCore Runtime Protocol Is Immutable and Validated at Plan Time

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-53 · **Source PRD:** PRD-007 · **Status:** Accepted — mechanism corrected by AD-103 · **Related:** AD-4, AD-52, AD-103

> **Correction (2026-06-28, AD-103):** the "immutable / destroy-and-recreate" and "caught at plan time" claims below are **factually wrong as-built.** `update_agent_runtime` is full-replace, so `serverProtocol` is **mutable in place** (it was flipped MCP→A2A→MCP on the same skill runtime id, no recreate), and the skill runtime is updated **out-of-band via boto3** — a path the plan-time TF check never sees. The intent (agents=A2A, skills=MCP) still holds; the *protection mechanism* is superseded by the guarded re-assertion path in **AD-103**.

## Context

AgentCore Runtimes are typed at creation: agent Runtimes use A2A (port 9000) and Skill Runtimes use MCP (port 8000, path `/mcp`). The `server_protocol` field in `aws_bedrockagentcore_agent_runtime` cannot be changed after creation — deploying the wrong value requires destroy-and-recreate of the Runtime. A mismatch between the configured protocol and the actual invocation method causes silent invocation failures that are difficult to diagnose after deployment.

## Decision

Treat the runtime protocol as immutable and validate it at Terraform plan time via a policy check, catching mismatches before any resource is created or modified.

## Alternatives Considered

- **Discover protocol mismatches at deploy time via integration tests.** Rejected: destroy-and-recreate of a Runtime is expensive and disruptive; catching the error before resource creation is strictly cheaper.
- **Document the constraint in a runbook and rely on operator discipline.** Rejected: silent invocation failures are hard to attribute to a protocol mismatch without tooling; a plan-time check is a zero-cost enforcement point.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A class of expensive, hard-to-diagnose silent invocation failures is caught at plan time before any resource is touched | Nothing material — the only cost is the plan-time policy check and the discipline of never editing the field in place |

## Results

All six agent Runtimes (AD-4; seven before AD-117) are configured with `server_protocol = "A2A"`, as are their canary siblings (AD-131); the tenant-agnostic `skill-runtime` image uses `server_protocol = "MCP"`. This constraint is codified alongside the 2 GB ARM64 image requirement and the 15-minute `InvokeAgentRuntime` ceiling as immutable platform facts the deployment must respect. The plan-time check is applied via the Terraform policy check step in CI/CD (REQ-I004). AD-52 documents the broader provider-coverage context in which the Runtime resource lives.

**Update 2026-08-19 (impl PR #335) — the protocol-mismatch risk this decision names also lives on the caller side, which neither AD-053's plan-time check nor AD-103's guarded re-assertion path guards.** Both cover whether a Runtime's *own* `serverProtocol` is configured correctly; neither checks whether a *caller* shapes its request for the protocol the target Runtime actually speaks. `orchestrator/warm_runtimes.py` pinged every deployed Runtime with one A2A `message/send` envelope on `accept: application/json` — correct for the six A2A agents, but a 406 for the four MCP servers (`dev_skill_runtime`, `dev_step_functions_orchestrator`, `dev_dynamodb_master_data`, `dev_tenant_mdm_emulator`), rejected by MCP's streamable-HTTP content negotiation before the container is even reached. That ping never warmed those four Runtimes and tripped `dev-buyer-team-skill-runtime-agent-errors` on a fixed cadence — the "silent invocation failure... difficult to diagnose after deployment" this ADR named, now manifesting in a client instead of in Terraform. `_agent_runtimes()` now calls `GetAgentRuntime` to read each Runtime's actual `serverProtocol` before shaping the ping (A2A `message/send` vs MCP `initialize`, with the matching `Accept` header), so protocol correctness is looked up per Runtime rather than assumed once for the whole fleet.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
