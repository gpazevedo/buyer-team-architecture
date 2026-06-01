# AD-074 — Four Mem0 Integration Points, All Degrade Gracefully

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-74 · **Source PRD:** PRD-014 · **Status:** Accepted · **Related:** AD-72, AD-46, AD-20

## Context

Cross-negotiation memory could be wired into many agent call sites. Without an explicit decision on which integrations are worth the Mem0 dependency, each agent team might add their own, producing an untested and inconsistently degrading set of call sites. The critical constraint from AD-46 is that no Mem0 integration may block a negotiation; every integration point must degrade gracefully. The question is which four points deliver the most decision-quality improvement for the dependency cost.

## Decision

Wire exactly four Mem0 integration points: supplier relationship history (IP-1, highest impact, used by strategic and bottleneck agents), cross-negotiation bid patterns (IP-2), approver preference profiles (IP-3), and communication effectiveness (IP-4). Every integration point degrades gracefully — none is ever a hard dependency. A Mem0-specific circuit breaker opens after 5 consecutive failures within a 15-minute window; while open, each integration point skips Mem0 and returns empty, relaxing quality thresholds via `memory_degraded` (AD-20) but never blocking a negotiation.

## Alternatives Considered

- **Wire Mem0 into every agent that could benefit.** Rejected: untested degradation paths, higher coupling, and unclear ROI on low-impact agents; the four chosen points cover the highest-value decisions.
- **Wire only IP-1 (supplier history) in v1.0.** Rejected: IP-2 through IP-4 can be implemented alongside IP-1 with the same circuit-breaker infrastructure already in place; deferring them adds a future integration cost without reducing current complexity.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Concrete decision-quality improvements at the highest-value points — strategic and bottleneck agents get real supplier history instead of `[]` | Guaranteed availability of the enhancement — a negotiation can proceed with degraded inputs (relaxed thresholds, no historical context) |
| The never-block rule keeps Mem0 strictly a quality enhancer, not a correctness dependency | Decision quality varies silently with Mem0 health unless the `memory_degraded` signal, relaxed-threshold guards, and no-auto-award-below-relaxed-thresholds are all operating correctly |
| One circuit breaker instance covers all four integration points, limiting cascading failures | The circuit breaker protects all four IPs together — a Mem0 failure in any one IP opens the breaker for all |

## Results

Each integration point has a defined writer agent and reader agent. The existing `RelationshipHistoryEnforcement` steering hook now receives real supplier history data from IP-1 instead of an empty list. Degradation is observable via `memory_degraded` (AD-20) and bounded by the 5-failure / 15-minute circuit breaker. This is consistent with AD-46's never-block rule, which is enforced on both the write and read paths of each IP. The two-tier memory architecture that makes these IPs possible is defined in AD-72.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
