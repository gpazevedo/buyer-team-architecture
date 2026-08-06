# AD-136 — Guardrail-Triggered Reconnaissance Detection (REQ-S601)

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-136 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-43, AD-128, AD-36, AD-24

## Context

MITRE ATLAS's Reconnaissance tactic (AML.T0013) covers an adversary probing a system to map its defenses before staging a real attack — here, a supplier sending a sequence of near-miss prompt-injection payloads to learn what Bedrock Guardrails blocks, then crafting an optimized payload offline before submitting it live. PRD-005's v1.1.0 ATLAS uplift specified a detection control for this (REQ-S601: alarm on `security.guardrail.triggered` bursts) but, like several other controls from that uplift (REQ-S605 GuardDuty, REQ-S606 Automated Reasoning, REQ-S607 contextual grounding), it was designed and never built — confirmed absent by direct `infra/` and `impl/` inspection during the 2026-08-06 PRD-005 reconciliation. Unlike those three, this one was cheap to close: the signal (a genuine Guardrail block) was already being computed at the point `GuardrailBlocked` is raised; it just wasn't being emitted as a metric.

## Decision

Emit `security.guardrail.triggered` (namespace `procurement/resilience`, dimensions `{tenant_id, agent_name}`) from `buyer_agent_core.serve._emit_guardrail_triggered` at both raise sites — the non-streaming `run()` path and the streaming executor's — immediately before `GuardrailBlocked` propagates. The emission reuses AD-128's `is_masking_only_intervention` classification: only a genuine BLOCKED-policy stop counts, never a masking-only ANONYMIZE intervention, so routine PII redaction on legitimate negotiation content (the exact false-positive AD-128 exists to prevent) does not pollute the reconnaissance signal.

A new CloudWatch alarm, `guardrail_triggered_recon`, fires when the Metrics Insights sum of this metric exceeds 5 in a 60-minute window, wired to the existing `evaluation_alerts` SNS topic (AD-24's subscriber, when live). The aggregation is deliberately coarse: `SUM(...) FROM SCHEMA(..., tenant_id, agent_name)` with no per-entity `GROUP BY`, summed across *all* tenants and agents in the window — matching every other alarm in `bucket_a_alarms.tf`. This is a scope narrowing from REQ-S601's literal per-supplier wording: no request shape in the system is uniformly single-supplier (`SpotBidRequest` carries a supplier list, for instance), while `tenant_id`/`agent_name` are the dimensions universally available at the block-detection point regardless of request shape.

## Alternatives Considered

- **Per-supplier gating, as REQ-S601's literal wording specifies.** Rejected: no request shape in the system is uniformly single-supplier, so a per-supplier dimension isn't available at the emission point without redesigning multiple request schemas for one alarm's benefit.
- **A new namespace (`procurement/security`, as originally specified) instead of reusing `procurement/resilience`.** Rejected in favor of consistency: every sibling alarm in `bucket_a_alarms.tf` already lives in `procurement/resilience`, and splitting one metric into a new namespace would cost a second Metrics Insights schema for no detection benefit.
- **Fire the metric from the Guardrail's own trace inspection, independent of the `GuardrailBlocked` raise.** Rejected: the raise sites are already the single point where "genuine block, not masking" has been determined (AD-128); duplicating that classification elsewhere risks the two paths drifting.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A real, applied signal for the Reconnaissance tactic that ATLAS's threat model calls for — closes a gap that sat undetected in PRD-005 for months | Aggregate, cross-tenant/cross-agent threshold means a patient adversary who spreads probing traffic across enough tenants or agents, or paces it under 5/hour, stays under the alarm |
| Reuses AD-128's masking classification, so the signal doesn't fire on routine PII anonymization — no new false-positive surface | One more alarm dependent on `evaluation_alerts` having a live subscriber (AD-24's open item) to actually page anyone |
| Cheap to build — the classification and raise sites already existed; this is one helper function and one alarm resource | Doesn't detect the ML Attack Staging phase that follows reconnaissance (payload crafted offline) — PRD-005 §1.3 still rates that tactic Partial |

## Results

Shipped and applied to dev (impl PR #263, 2026-08-06): `_emit_guardrail_triggered` in `packages/buyer_agent_core/buyer_agent_core/serve.py`, wired into both the `run()` and streaming-executor raise sites; `aws_cloudwatch_metric_alarm.guardrail_triggered_recon` in `infra/bucket_a_alarms.tf`, applied via a scoped `terraform apply -target` (chosen specifically to avoid bundling in an unrelated pre-existing GitSHA tag drift on the SNS topic). Tests in `test_serve_guardrail_block.py` and `test_streaming.py` assert the metric fires on a genuine block and does not fire on a masking-only ANONYMIZE intervention. PRD-005 §1.3's Reconnaissance row and §10.7's REQ-S601 entry are updated Coverage None → Covered.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
