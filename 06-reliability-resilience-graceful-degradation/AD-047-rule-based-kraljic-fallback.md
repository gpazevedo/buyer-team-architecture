# AD-047 — Rule-Based Kraljic Fallback with Quadrant-Specific Rejection

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-47 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-5, AD-46

## Context

Routing at Node 3 depends on the Kraljic Classifier agent. If that agent is unavailable or returns malformed output after retries, the negotiation cannot proceed — but a wrong automated classification on a high-stakes quadrant is dangerous. The LLM-based classifier achieves approximately 95% accuracy; a rule-based fallback using the same system-config thresholds achieves 75–90% depending on quadrant. Proceeding automatically with 75% accuracy on BOTTLENECK or STRATEGIC items risks irreversible procurement decisions on misclassified high-risk goods.

## Decision

When the Kraljic Classifier agent is unavailable, fall back to deterministic rule-based classification using the same `{env}-system-config` thresholds. Apply quadrant-specific rejection criteria: escalate the risky fallback cases to REQUIRES_ATTENTION rather than auto-proceeding. NON_CRITICAL is always accepted (safe to auto-award); LEVERAGE with fewer than two suppliers, BOTTLENECK with an empty approved pool, and STRATEGIC with three or more risk flags or no supplier history are escalated to human review.

## Alternatives Considered

- **Halt the negotiation (REQUIRES_ATTENTION) on any classifier unavailability.** Rejected: needlessly blocks low-risk items that a deterministic rule can safely classify; violates the availability-first principle of AD-46.
- **Auto-proceed with rule-based classification for all quadrants.** Rejected: the fallback is only ~75% accurate for BOTTLENECK, making automated decisions on misclassified high-risk items unacceptably dangerous.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Low-risk (NON_CRITICAL) work keeps flowing during classifier outages | Accuracy drops to 75–90% vs the LLM's ~95%; the design compensates by rejecting the dangerous fallback cases into human review |
| Throughput is preserved exactly where the fallback is strongest | BOTTLENECK, LEVERAGE (low-supplier), and STRATEGIC (high-risk) items incur human-review latency during classifier outages |
| Fallback decisions are idempotency-keyed like any other side effect | Rule-based path bypasses LLM content filtering (though Cedar and partition isolation remain active per PRD-005 §1.4) |

## Results

Fallback decisions are idempotency-keyed like any side effect; rejected fallback cases set REQUIRES_ATTENTION. The quadrant-specific rejection thresholds are sourced from `{env}-system-config` alongside the primary classifier thresholds, so they tune together. Security note: the rule-based path bypasses LLM content filtering but never bypasses Cedar or partition-key isolation (PRD-005 §1.4). This decision applies the availability-first principle of AD-46 to the Kraljic routing node specifically, and uses the same REQUIRES_ATTENTION escalation path defined in AD-16.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
