# AD-001 — Adopt a Hybrid Two-Level Topology: Orchestration Before Intelligence

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-1 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-11, AD-12, AD-18

## Context

Buyer Team automates a governed, audited business process (procurement negotiation) using LLM agents. LLM control flow is non-deterministic, but procurement requires reproducible execution paths, enforceable policy guardrails, and a complete audit trail (OKR O4: 100% governance compliance, 100% audit-trail completeness). A purely agentic design — where an agent decides its own next steps — cannot guarantee any of these. The system must therefore assign, explicitly, who controls the workflow shape versus who supplies adaptive intelligence.

## Decision

Adopt a hybrid two-level topology. **Level 1** is a deterministic AWS Step Functions DAG that defines the workflow's structural backbone and records every state transition. **Level 2** is a set of LLM agents (Strands A2A on AgentCore Runtime) that supply adaptive intelligence *at each node*, bounded by guardrails. Intelligence is invoked by the orchestrator — never the other way around.

## Alternatives Considered

- **Fully autonomous agent orchestration (agent-as-planner).** Rejected: non-deterministic control flow is incompatible with the audit and governance KRs; a misbehaving agent can alter the workflow shape in ways that cannot be traced or reproduced.
- **Static rules-only automation (no LLM agents).** Rejected: cannot handle the variability of supplier negotiations, qualification criteria, or category-specific strategy adaptation across the Kraljic quadrants.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Deterministic, replayable control flow and complete state-transition history | Workflow topology is fixed — a new negotiation style requires a DAG change, not just a prompt change |
| Governance and approval gates enforced by DAG structure rather than prompt text | Two control planes (Step Functions + AgentCore) must be operated and reasoned about together |
| A misbehaving agent cannot alter the workflow shape | Intelligence at each node must fit the pre-defined node boundaries |

## Results

Realized as the seven-node DAG in PRD-002 (AD-11) with a single governed cycle-back (AD-12), and governance enforced in deterministic code at the node level rather than via prompts (AD-18). This is the root decision the rest of the architecture hangs from — it is what makes the 100% governance-compliance and 100% audit-completeness targets achievable.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
