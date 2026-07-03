# AD-034 — Evaluation Score Thresholds Drive Automated Actions (Closed Loop)

**Theme:** Observability & Evaluation  **Catalog:** AD-34 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-30, AD-32, AD-33, AD-35

## Context

Detecting quality drift is only half the problem; the system must act on it without waiting for a human in the common cases. Without a binding consequence for each evaluator's score, quality drift degrades silently. Compliance-critical violations (confidentiality, winner disclosure, governance) require immediate escalation rather than deferred review. During memory degradation, the baseline quality threshold must relax to avoid false alarms triggered by reduced evaluation completeness rather than genuine model regression.

## Decision

Bind each evaluator to a minimum score, a CloudWatch alarm threshold, and an automated action wired through CloudWatch alarms → SNS:

- Negotiation Quality Composite < 0.60 → model rollback.
- Communication Tone < 0.80 → disable auto-send.
- Rationale Defensibility < 0.70 → block auto-award.
- Kraljic Accuracy < 0.90 → invalidate semantic cache and alert.
- Bid Format / Governance Compliance < 1.00 → alert immediately / halt new negotiations.

Confidentiality and winner-disclosure violations escalate to Compliance on a single event. During memory degradation, the quality alarm threshold relaxes to 0.85 (PRD-004 §2.3.2 / REQ-O209) to avoid false alarms caused by reduced evaluation completeness rather than genuine model regression.

## Alternatives Considered

- **Thresholds drive alerts only, no automated actions.** Rejected: alert-only requires a human to react to every quality drift event; this latency is unacceptable for events that block negotiations or require model rollback.
- **Single global quality threshold with no per-evaluator binding.** Rejected: different evaluators measure different failure modes with different consequences; a single threshold cannot distinguish a tone regression (disable auto-send) from a governance violation (halt + compliance escalation).
- **Status quo / no action.** Rejected: without binding consequences, quality drift degrades silently until a human notices.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Quality control is closed-loop and largely hands-off for routine drift; the system can roll back models, disable auto-send, block auto-award, and halt new work without manual intervention | A flaky or noisy evaluator could trigger automated rollback or halt, over-reacting to a transient signal |
| Compliance-critical violations (governance, confidentiality, winner disclosure) still escalate immediately to humans regardless of automation | Online evaluation alarms run over multi-minute periods (e.g., 15 min), so some sub-threshold output can ship before the automated action fires |
| Degraded-mode threshold relaxation prevents false alarms during memory degradation | Governance and format compliance alarm at exactly 1.0 — any single violation halts or alerts, which may interrupt throughput for a genuine edge case |

The LLM-as-Judge consistency requirement (±0.1 across three runs) mitigates the false-positive risk from flaky evaluators. The bounded detection lag is an explicitly accepted trade-off in favor of managed-service simplicity over a custom real-time pipeline.

## Results

Routine quality drift is handled automatically without human intervention. Compliance-critical violations escalate immediately to humans. The accepted liabilities are eval-driven false positives and a bounded detection lag of up to one alarm period. AD-30 (Transaction Search) is the prerequisite for evaluators to have input; AD-32 defines the evaluator types; AD-33 defines the sampling strategy that determines which traces contribute scores. AD-35 extends the closed loop with scheduled ATLAS-specific evaluators whose failures also gate deployment.

**Not built (as of PR #108, 2026-07-02).** None of the five CloudWatch-alarm-driven automated actions described above (model rollback, disable auto-send, block auto-award, invalidate semantic cache, halt on governance/format violation) have been observed wired to CloudWatch alarms or SNS. This closed loop is only as real as the evaluator scores feeding it, and per AD-32/AD-33's Results only the Kraljic Accuracy ground-truth check is implemented and CI-gated today — offline, not against live alarms. REQ-O223 (threshold re-baselining trigger) is implemented as a negotiation-count countdown persisted in `{env}-system-config`, published via `resilience.metrics.emit_metric` rather than OTEL (the orchestrator Lambdas' ADOT layer is trace-only — see [AD-031](AD-031-w3c-traceparent-propagation.md)'s Results for why that layer was dark until the same day). This ADR's decision stands as the target design; treat every action in the Decision section as unimplemented until an individual Results update says otherwise.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
