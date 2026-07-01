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

All seven agent Runtimes (AD-4) are configured with `server_protocol = "A2A"`; the tenant-agnostic `skill-runtime` image uses `server_protocol = "MCP"`. This constraint is codified alongside the 2 GB ARM64 image requirement and the 15-minute `InvokeAgentRuntime` ceiling as immutable platform facts the deployment must respect. The plan-time check is applied via the Terraform policy check step in CI/CD (REQ-I004). AD-52 documents the broader provider-coverage context in which the Runtime resource lives.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
