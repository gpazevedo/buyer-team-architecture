# AD-156 — DLQ Recovery Is a Three-Tier Model, Factored into One Reusable Consumer

**Theme:** Reliability, Resilience & Graceful Degradation
**Catalog:** AD-156 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-45, AD-69

## Context

Before this decision, the codebase had exactly one DLQ consumer — `resilience/redrive.py`, which resumes a negotiation from its checkpoint (AD-45's "retry exhaustion routes to DLQ" leg) — and one DLQ with an alarm and no consumer at all: `{env}-buyer-team-po-export-dlq`. An awarded order whose SAP export failed five times (`po_export_drain`'s own `maxReceiveCount=5`) landed there and stopped for good, with no automated path back to `REQUIRES_ATTENTION`. Writing a second bespoke Lambda to fix that gap would have left the classify/bound/escalate loop duplicated between two consumers that differ only in what they replay and where they escalate to — and a third was already anticipated: PRD-011 REQ-M504's Kafka dead-letter *topic* for schema-invalid events has no consumer either, `skills/shared/kafka_transport.py` being unbuilt.

Separately, PRD-011 §7 ("Kafka broker unavailable → buffer events in SQS dead-letter queue") and REQ-M504 (§9.5: "Kafka consumer schema validation failure → reject event, publish to dead-letter topic `{env}.{tenant_id}.{domain}.dlq`") read as two conflicting descriptions of where a failed Kafka event ends up — a queue in one place, a topic in another — with nothing in either section saying they are not alternatives.

## Decision

Recovery from failure is a three-tier model, and each tier gets exactly one mechanism:

1. **Bounded in-Lambda retry** — a handful of attempts with backoff inside one invocation (`po_export_drain._with_retries`, the A2A retry wrapper). Catches a transient blip without ever touching a queue.
2. **SQS `maxReceiveCount` redrive** — the queue's own poison-message mechanism moves a message that exhausted its natural redeliveries into a dead-letter queue. This is the *broker/queue-unavailable* buffer: nothing about the message's content was rejected, delivery just couldn't complete.
3. **Second-level requires-attention escalation** — a dedicated consumer classifies each DLQ message, replays what might still succeed, and routes what can't to `{env}-buyer-team-requires-attention-dlq` *and* a durable write onto the record's own DynamoDB row (`export_status=REQUIRES_ATTENTION` for orders, `status=REQUIRES_ATTENTION` for negotiations) — the row, not the fire-and-forget SQS message, is authoritative (AD-45's correction note).

Tier 3's classify/bound/escalate loop is factored once into `resilience/dlq_consumer.consume_dlq(receive_fn, replay_fn, classify_fn, escalate_fn, *, max_redrives, max_messages)`, transport- and domain-agnostic by construction: `receive_fn`/`escalate_fn` are the only transport-specific pieces, `replay_fn`/`classify_fn` the only domain-specific ones. `orchestrator/po_export_redrive.py` is its first driver, supplying SQS receive/delete for `PO_EXPORT_DLQ_URL`, `po_export_drain.export_to_sap` as replay, and a classifier mapping `SapTerminalError`/`SapSupplierUnmapped`/`UnknownTargetSystem`/`KeyError` to escalation reasons. `resilience/redrive.py` is *not* ported onto it in the same change — landing one driver first keeps a regression in the negotiation DLQ isolable from the new po-export consumer. A future Kafka DLQ-topic consumer (REQ-M504) becomes a third driver by supplying only its own `receive_fn`/`escalate_fn` — `receive_fn` polling the topic instead of an SQS queue, `escalate_fn` writing wherever that domain's second-level surface is — with the same `classify_fn`/`replay_fn` split.

The PRD-011 §7/§9.5 tension is not a contradiction: they describe tier 2 and a domain-specific instance of "reject a bad message," respectively, at two different points in the pipeline. §7's SQS buffer is what happens when the *broker itself* is unreachable — nothing was ever received. REQ-M504's dead-letter topic is what happens when a message *was* received and failed schema validation — the Kafka-native poison-message surface, upstream of any SQS queue. Both PRD-011 §7 and §9.5 are updated in place to say so explicitly.

## Alternatives Considered

- **A bespoke `po_export_redrive.py` with its own copy of the classify/bound/escalate loop.** Rejected: `redrive.py` already proved this loop is identical shape across domains (receive → classify → bound-by-receive-count → escalate); copying it a second time guarantees the two consumers drift the first time one gets a bug fix the other doesn't.
- **Port `redrive.py` onto `consume_dlq` in the same change that introduces it.** Rejected: `consume_dlq` is unproven with zero drivers; migrating the one working consumer onto a new, thinly-tested abstraction at the same time removes the ability to isolate a regression to either the abstraction or the migration.
- **Treat PRD-011 §7 and REQ-M504 as literally conflicting and pick one.** Rejected: they describe genuinely different failure points (broker-unreachable vs. received-but-invalid); picking one would silently drop the other's failure mode instead of building both.
- **Build the Kafka broker now to give `consume_dlq` a real third driver immediately.** Rejected: no broker, no standing MSK cost — this decision documents the pattern's shape so a future Kafka consumer slots into it, not a reason to stand up infrastructure early.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One classify/bound/escalate implementation instead of N; a Kafka DLQ-topic consumer needs only its own `receive_fn`/`escalate_fn` | `consume_dlq` ships proven against exactly one driver (`po_export_redrive`) — its generality over a genuinely different transport (Kafka offsets vs. SQS receipt handles) is a design bet, not yet demonstrated |
| The po-export DLQ gap (alarm with no consumer) is closed using the same authoritative-row invariant `redrive.py` already established, rather than inventing a second convention | `redrive.py` itself is untouched — the negotiation DLQ does not yet benefit from the shared engine, so a future migration is still open work |
| PRD-011 §7/§9.5 no longer read as contradictory to a new reader | Two dead-letter surfaces (SQS queue, Kafka topic) still exist for two different failure modes — the fix is documentation, not a reduction in moving parts |

## Results

`resilience/dlq_consumer.py` (`consume_dlq`, `DlqMessage`) ships with `orchestrator/po_export_redrive.py` as its first driver, scheduled every 5 minutes against `{env}-buyer-team-po-export-dlq`. `po_export_drain.export_to_sap` gained an `UnknownTargetSystem` exception (case-insensitive `target_system` match) so a non-`sap` export — previously silently dropped as `"skipped"` — is now itself a redrive-classifiable failure instead of a message the drainer just deletes. New alarm `${env}-buyer-team-po-export-redrive-exhausted` on the `po_export_redrive_exhausted` metric, and a dashboard widget alongside the existing DLQ-redrive one. PRD-011 §7 and §9.5 (REQ-M504) both carry an explicit cross-reference to this ADR. Migrating `resilience/redrive.py` onto `consume_dlq`, and building the Kafka DLQ-topic consumer as `consume_dlq`'s second real transport, are both follow-on work this ADR sets up but does not itself do.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
