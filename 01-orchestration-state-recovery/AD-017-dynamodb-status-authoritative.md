# AD-017 — DynamoDB Status Write Is Authoritative; DLQ Best-Effort + S3 Archive

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-17 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-16

## Context

Escalation to REQUIRES_ATTENTION must be reliable. If the escalation signal depended on SQS, an SQS/DLQ outage could silently lose escalations — a negotiation would fail without anyone being told. Separately, the DLQ's native 14-day retention is too short for the audit horizon of a failed financial negotiation.

## Decision

Make the DynamoDB status write the single authoritative escalation signal: on retry exhaustion, write `Negotiation.status = REQUIRES_ATTENTION` with `entry_trigger`/`entry_reason`, then raise the CloudWatch alarm. The SQS DLQ publish is fire-and-forget for ops tooling only, decoupled so it can *never* block the status write (REQ-G306). Each DLQ publish is additionally tee'd to an immutable S3 archive (`{env}-buyer-team-dlq-archive`, Object Lock GOVERNANCE, 7-year retention); that archive write is likewise decoupled and best-effort.

## Alternatives Considered

- **DLQ as the primary escalation signal.** Rejected: couples escalation to SQS availability; an outage loses escalations.
- **Synchronous, coupled status + DLQ write.** Rejected: a DLQ failure would roll back or block the status write, defeating the purpose.
- **DLQ retention only, no S3 archive.** Rejected: 14 days is insufficient for audit.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Escalation is never silently lost — the status write succeeds even when SQS/S3 are unavailable | Eventual consistency between the authoritative status and the ops DLQ message — operators may see REQUIRES_ATTENTION before the DLQ message arrives |
| 7-year audit trail for failed negotiations beyond the DLQ's 14-day retention | Storage cost and a second best-effort write per escalation for the S3 archive |

## Results

A DLQ or S3 outage degrades only ops tooling and long-term audit convenience, never the escalation itself, which is always visible in CloudWatch and the dashboard. SQS publish failure emits `procurement/graph/dlq_publish_failed` independently. Failed-negotiation records remain auditable for seven years beyond the DLQ's 14-day window.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
