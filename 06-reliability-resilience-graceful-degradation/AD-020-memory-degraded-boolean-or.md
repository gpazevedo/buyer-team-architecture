# AD-020 — `memory_degraded` Is the Boolean OR of Two Independent Signals

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-20 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-72, AD-13, AD-46

## Context

The platform runs two memory tiers that fail independently: AgentCore Memory (turn-by-turn, session scope) and Mem0 (cross-negotiation knowledge). Either can degrade to a fallback path mid-negotiation. If degradation were silent, quality thresholds calibrated for full memory would keep applying and the system would over-trust a degraded run. The orchestrator needs a single, cheap signal to consult after each agent call (REQ-G282).

## Decision

Represent degradation as a single `Negotiation.memory_degraded` boolean computed as the OR of two independent ContextVar signals: `MemorySignal` (AgentCore Memory, session scope) and `Mem0DegradationSignal` (Mem0, cross-negotiation scope). The flag is initialized `false` at Node 3, read from the A2A response payload after each strategy-agent node (4a–4d), and written before checkpoint. It is cleared back to `false` only when both signals have recovered. Memory degradation never blocks a negotiation (graceful degradation per AD-46).

## Alternatives Considered

- **Track each memory tier's status separately downstream.** Rejected: every consumer would have to know both signals and re-OR them; one flag is simpler and harder to misread.
- **Treat any degradation as a hard failure.** Rejected: violates graceful degradation — memory is a quality enhancement, not a hard dependency (AD-46).
- **Latch the flag once set and never clear it.** Rejected: a transient blip would pin the negotiation to degraded thresholds for its whole life, even after both tiers recover.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One flag, one read site — consumers never re-implement the OR logic | Conservative over-flagging when only the less-critical tier is down |
| Quality thresholds can renormalize mid-negotiation once both tiers recover | Per-node bookkeeping and a symmetric recovery path on each tier's adapter |
| Degraded runs are visible to post-completion evaluators (PRD-004 REQ-O209) | Consumers cannot distinguish which tier caused the degradation from the flag alone |

A single OR is conservative — any one tier degrading flags the whole negotiation — but that is the safe direction: it never under-reports degradation.

## Results

Quality thresholds renormalize as soon as both memory layers recover, rather than after the negotiation ends. Degradation is surfaced on a single read site — the wrapper response of AD-13 — and never halts a negotiation. The flag persists through later nodes and feeds post-completion quality evaluation (PRD-004 REQ-O209), so degraded runs are visible to evaluators. This flag is the signal consumed by the no-auto-award guard defined in AD-46.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
