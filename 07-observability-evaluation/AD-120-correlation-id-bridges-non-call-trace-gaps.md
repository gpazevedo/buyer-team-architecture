# AD-120 — Correlation ID Persisted as Row Data to Bridge Trace Discontinuities Beyond A2A Calls

**Theme:** Observability & Evaluation  **Catalog:** AD-120 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-29, AD-31, AD-13, AD-77

## Context

AD-31's W3C `traceparent` propagation reconnects every hop that is a live A2A call, enforced through the shared `invoke_agent_runtime` wrapper (AD-13), and AD-29 already flagged that a negotiation spanning three observability layers "requires a correlation mechanism... to recombine." But two hops in the PR→PO lifecycle are not live calls at all, so there is no header to propagate on them:

1. **PR ingest.** A PR is created in the master data store, then picked up by `pr_event_router` off a **DynamoDB Stream record** (AD-77) — a transport with no concept of a header or an in-flight call to carry one on.
2. **The human-approval pause.** `node_approval_gate` blocks on a Step Functions `waitForTaskToken` that can sit open for an arbitrary real amount of time — minutes to days — until a human calls the separate `/approve` API. The API call that resumes the wait is a brand-new process invocation with no trace context of its own; there is nothing "in flight" to extract a `traceparent` from.

Without a mechanism for these two gaps, the single coherent per-negotiation trace AD-29/AD-31 promise breaks exactly at PR creation and exactly at the highest-latency, most operationally interesting step (human review) — the two places an operator or auditor is most likely to go looking.

## Decision

Generate a `correlation_id` (UUID4, today only by the demo harness's `pr_generator.py`) at PR-creation time and persist it as a plain attribute — `graph_nodes.correlation_id` — on the PR row in the master data store, not as a transport header. `pr_event_router` reads it directly off the DynamoDB Stream record's PR item and seeds it into the Step Functions execution input, from which every node forwards it via the `carry_correlation_id` helper (`trace_helpers.py`) as a `procurement.correlation_id` span attribute.

At the approval-gate pause specifically, `_pause_for_human` persists **both** `correlation_id` and the injected OTEL W3C carrier (`otel_ctx`) as attributes on the `NEGOTIATIONS_TABLE` row. On resume, `lambda_handler` detects the resume payload has no `_otel_ctx` of its own, reads `_persisted_trace_fields()` back off that row, and merges it into the payload before extracting a `Context` — so the post-approval span nests under the pre-pause span instead of starting a disconnected trace. The general pattern is the same in both cases: where there is no live call to carry a header on, persist the correlation state as plain row data and restore it explicitly at the next invocation, rather than only ever forwarding it in-band.

## Alternatives Considered

- **Serialize the OTEL `SpanContext` itself into the Step Functions task-token payload and rely on that alone.** Rejected: `otel_ctx` already is the serialized carrier, and persisting it durably on the negotiation row rather than only in the task-token payload was necessary regardless — the resuming caller (the approval API) never has the task-token payload to hand back. Doing this without also carrying a plain, business-readable `correlation_id` would leave the demo harness, frontend SSE, and audit views with no identifier to reference short of parsing an OTEL carrier.
- **Key correlation off the Step Functions execution ARN.** Rejected: the execution ARN doesn't exist yet at PR-creation time (before the execution starts) and doesn't reach backward across the DynamoDB Stream → `pr_event_router` hop that precedes it.
- **Status quo / accept disconnected traces at these two hops.** Rejected: defeats AD-29's stated goal of one coherent negotiation view precisely where it matters most for auditing a human decision.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The PR→PO trace survives both the DynamoDB-Stream ingest hop and an arbitrarily long human-approval pause, not just live A2A calls | Two parallel correlation mechanisms to keep behaviorally consistent — `traceparent` for live calls (AD-31), row-persisted `correlation_id`/`otel_ctx` for durable/non-call gaps — rather than one |
| A single human-readable identifier threads through DynamoDB → Stream → SFN input → span attributes → demo harness SSE/API responses | Restoring trace context after a pause depends on remembering to persist it at every future pause point; a node that blocks without following `_pause_for_human`'s pattern would silently reopen the gap |

`correlation_id` is currently generated only by the demo harness's `pr_generator.py`; `master_data_client.create_pr`'s `correlation_id` parameter defaults to `None`, and when omitted, `graph_nodes` gets no `correlation_id` key at all. Real-tenant PRs ingested via the two production MCP servers (AD-77) do not currently get one — this decision's mechanism is proven end-to-end for the demo tenant only, not yet wired to the real ingestion path.

## Results

Shipped across PR #160 (PR-creation → SFN → node span attribute), #161 (demo harness generation + SSE/state surface), and #164 (persist/restore across the approval-gate pause, plus a fix to `_send_success` that had been shipping output to Step Functions before trace-context injection could run — silently breaking continuity on *every* task-token resume, not only human-reviewed ones; see AD-31's update). `orchestrator/node_approval_gate.py`'s `_pause_for_human` / `_persisted_trace_fields` / `lambda_handler` is the reference implementation for "persist both identifiers as row data, restore at the next invocation" — any future node that introduces its own durable pause should follow the same pattern rather than assume `traceparent`'s live-call propagation (AD-31) is sufficient on its own. Wiring `correlation_id` generation into the real-tenant MCP ingestion path (AD-77) so this isn't demo-only remains open.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
