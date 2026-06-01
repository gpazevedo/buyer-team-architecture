# AD-077 — Two New MCP Servers and Event-Driven PR Pickup via DynamoDB Streams

**Theme:** Test Tenant Platform & Data
**Catalog:** AD-77 · **Source PRD:** PRD-015 · **Status:** Accepted · **Related:** AD-75, AD-76, AD-78

## Context

The test tenant must emulate the shape of a real ERP integration: an external system that owns master data and emits events when purchase requisitions are created or cancelled, rather than writing directly into Buyer Team's domain tables. Without this shape, the demo validates a shortcut integration path that no production tenant would use. Additionally, a realistic inbound trigger is needed when a PR is created in the master store — both by the loader (seeded PRs) and by the PR Simulator in the Test Tenant App (PRD-013).

## Decision

Add two new MCP servers: `dynamodb-master-data` (internal CRUD against the four master tables, scoped to the master store only) and `tenant-mdm-emulator` (an ERP-shaped external read API consumed by Buyer Team's Tenant Integration Skill). Drive PR pickup via a DynamoDB Stream on the master PR table that triggers a `pr-event-router` Lambda, which invokes Buyer Team's `ingest_purchase_requisitions` tool (gated by `auto_start_workflow`). The router re-fetches current PR state on every event rather than trusting the stream payload.

## Alternatives Considered

- **Write PRs directly into Buyer Team domain tables.** Rejected: bypasses the integration path a production ERP tenant would use, so the demo does not validate the real integration shape.
- **Polling the master store from Buyer Team on a schedule.** Rejected: introduces latency, adds operational complexity, and does not mirror the event-driven pattern of real ERP integrations.
- **Extend the existing `dynamodb-domain` MCP server to cover master tables.** Rejected: conflates internal domain state with the master store, increasing blast radius and violating the bounded-scope principle of MCP server design.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The test tenant exercises the same integration path a production ERP tenant would use — external read API plus event-driven inbound | More moving parts: two MCP servers, a DynamoDB Stream, a router Lambda, and a failure DLQ compared to direct domain writes |
| Stream-driven PR pickup mirrors real event-driven ERP integration | Two ingestion paths coexist temporarily — the legacy loader (AD-75) runs in parallel for shadow validation until its retirement is decided |
| The router's "re-fetch current state" invariant makes delivery at-least-once safe; `SENT_TO_BUYER → no-op` absorbs duplicate stream events | The no-op re-invocation triggered by `mark_pr_received` writing `SENT_TO_BUYER` is intentional overhead that must not be optimized away |

## Results

This decision brings the platform MCP-server total from 6 to 8 (the 6 production servers plus these 2 test-tenant-scoped ones, not used by production tenants). PR create and cancel flows through the master store and the Stream router. The `pr-event-router` Lambda is deployed with `MaximumRetryAttempts=3`, `BisectBatchOnFunctionError=true`, and an SQS DLQ (`{env}-pr-event-router-dlq`) with a CloudWatch alarm on DLQ depth > 0. The two ingestion paths consume the same CSVs and produce the same canonical entities; shadow comparison gates retirement of the legacy path (AD-75). The `tenant-mdm-emulator`'s ordered list tools depend on the synthetic GSI sort keys defined in AD-78.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
