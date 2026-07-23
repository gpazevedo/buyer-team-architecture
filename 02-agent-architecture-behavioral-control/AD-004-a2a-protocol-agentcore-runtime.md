# AD-004 — A2A Protocol; Each Agent Is Its Own AgentCore Runtime

**Theme:** Agent Architecture & Behavioral Control  **Catalog:** AD-4 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-13, AD-21, AD-27, AD-53

## Context

The system is decomposed into seven specialized agents, each owning one cognitive domain, invoked at defined nodes in the Step Functions workflow. They need to be independently versioned, tested, and replaced without redeploying the whole system. A deployment model must be chosen: a single multi-skill monolith, in-process calls between agents, or separate runtimes with a well-defined inter-agent protocol.

## Decision

Each agent is deployed as its own AgentCore Runtime and invoked over the A2A protocol (JSON-RPC 2.0, port 9000) via `InvokeAgentRuntime`. No agent shares a runtime with another; every invocation crosses a network boundary.

## Alternatives Considered

- **Monolithic multi-skill agent.** Rejected: couples all six agents' release cycles and removes per-agent independent testability and blast-radius isolation.
- **In-process tool calls between agents.** Rejected: agents cannot be independently versioned or replaced; a regression in one domain takes down the entire agent process.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Loose coupling — any single agent can be replaced, versioned, or tested in isolation | Inter-agent work crosses a network boundary, adding latency and additional runtimes to operate |
| Independent blast radius — a crash or bad deploy is scoped to one Runtime and one Kraljic branch | More failure modes to handle; resilience and cost attribution cannot be left to individual call sites |
| Per-agent model tiering, Cedar surface, and evaluation criteria become expressible | Consistent resilience patterns across all call sites require a shared wrapper (AD-13) |

## Results

This decision mandates AD-13 (a shared `invoke_agent_runtime` wrapper so resilience and cost attribution are uniform across every agent invocation). It enables AD-21 (single-responsibility decomposition of the seven cognitive domains) and AD-27 (agents communicate only through Step Functions shared state, never directly). The runtime protocol and ports are treated as immutable after creation, validated at Terraform plan time (AD-53).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
