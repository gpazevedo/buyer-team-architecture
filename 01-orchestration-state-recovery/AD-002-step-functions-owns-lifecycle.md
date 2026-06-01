# AD-002 — Step Functions Owns the Lifecycle; AgentCore Sessions Are Ephemeral

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-2 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-3

## Context

AgentCore sessions auto-terminate after 15 minutes of inactivity. A negotiation, however, can run for days or weeks (Strategic quadrant cycle target ≤ 28 days). The two timescales cannot be served by one mechanism, so lifecycle ownership must be assigned explicitly.

## Decision

The Step Functions *execution* owns the long-lived negotiation lifecycle. An AgentCore *session* is treated as a short-lived execution window only; the orchestrator opens a new session per agent invocation (each classification, bidding round, evaluation, or communication is discrete).

## Alternatives Considered

- **Use AgentCore sessions as the primary lifecycle holder.** Rejected: the 15-minute inactivity timeout cannot accommodate a negotiation that spans days or weeks; the platform's session semantics are fundamentally incompatible with procurement timescales.
- **Keep sessions alive with polling/heartbeats.** Rejected: fighting the platform's designed behaviour, adds complexity, and still cannot reliably sustain a weeks-long negotiation across network failures or restarts.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Design fits the platform's actual session semantics rather than fighting them | In-session continuity is absent — context cannot persist inside an agent across invocations |
| Recovery is clean because the durable lifecycle lives outside the agent runtime | Per-call context rehydration adds orchestration plumbing and per-call latency |
| A weeks-long negotiation decomposes into many minutes-long, independently recoverable agent calls | |

## Results

The 15-minute inactivity termination is accepted as designed-for behaviour; the orchestrator persists lifecycle state to DynamoDB checkpoints at each step (PRD-002, PRD-006 §4.1). Recovery resumes from the last completed node rather than from agent memory. This decision directly forces AD-3: if sessions are throwaway, durable state must live somewhere else.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
