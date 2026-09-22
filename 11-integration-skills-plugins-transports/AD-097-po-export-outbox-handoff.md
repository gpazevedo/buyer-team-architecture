# AD-097 — PO Export Decoupled from Award via a Durable Outbox

**Theme:** Integration, Skills, Plugins & Transports
**Catalog:** AD-97 · **Source PRD:** PRD-002 §3.7 / PRD-011 §3.2 · **Status:** Accepted (amended 2026-09-22) · **Related:** AD-9, AD-17, AD-67, AD-68, AD-158, AD-160, AD-161

## Context

Node 7 issues the Purchase Order (one per supplier, AD-9) and writes it authoritatively to `{env}-orders`. That PO still has to reach the tenant's system of record — a real tenant's ERP (SAP/Oracle/Coupa) over an export transport (AD-68), or, for the demo tenant, an internal MCP "PO Receiving" surface where the durable order row itself is the destination. The open question at award time is whether the orchestrator should call the export transport **inline**. Doing so couples award-completion latency and availability to the ERP and the transport: a transport outage would either stall the negotiation's COMPLETED transition or force a rollback of an already-authoritative PO. The export gap is invisible on the test tenant (internal PO Receiving) but is real for production tenants, so the seam has to be correct without an external transport actually existing yet.

## Decision

Decouple PO export from award completion through a durable outbox. After the authoritative `{env}-orders` write and the COMPLETED transition, Node 7 runs a **best-effort** handoff that never blocks COMPLETED:

- **No external target configured** (`{env}-tenant-skill-config{pk:tenant,sk:"po_export"}.target_system` absent — e.g. the test tenant's internal PO Receiving): the durable order row *is* the handoff; emit `po.export_internal` and call no transport.
- **Target configured** (real-tenant ERP): stamp the order `export_status=PENDING` + `export_target`/`export_format` (the outbox) and emit `po.export_requested`.

A separate transport drains the `PENDING` outbox asynchronously. Any error in the handoff is swallowed (`po.export_error`); the procurement decision and the PO stand.

> **Amendment, 2026-09-22.** As first written this sentence named "the Skill's `export_purchase_order` (PRD-011 §3.2, Phase 3)" as that transport. **No such function was ever built** — `grep -rn "def export_purchase_order" impl/` is empty, and `node_award_comms._resolve_po_export`'s docstring repeated the same claim until impl PR #467 corrected it. The real drainer is `po_export_drain`, an SQS consumer keyed on the order, and what it dispatches on is recorded below.

## Alternatives Considered

- **Inline synchronous ERP export at Node 7.** Rejected: couples award-completion availability and latency to the ERP and its transport; a transport outage would stall the COMPLETED transition or roll back a completed, authoritative PO.
- **No explicit handoff — a poller scans `orders` for un-exported POs.** Rejected: a blind scan cannot tell *whether* a PO needs export, to *which* target, in *which* format, without per-order intent; the outbox stamp records that intent explicitly and keeps export idempotent and observable.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Award completion is never blocked by ERP/transport health — the PO is authoritative the moment it is written | A second delivery stage (the drainer) must exist and be monitored; export is eventually-consistent, not immediate |
| Internal (test tenant) and external (real ERP) paths share one code seam; the demo path needs no transport at all | The `PENDING` outbox needs a drainer plus a retry / dead-letter story (Phase 3) before real tenants can export |

This mirrors AD-17's posture one layer out: the DynamoDB write is the authoritative event, and the outward delivery (DLQ there, ERP export here) is best-effort and decoupled so its outage can never block or corrupt the authoritative record.

## Results

Realized in `orchestrator/node_award_comms.py` (`_export_purchase_order`), invoked downstream of the durable `{env}-orders` write and the COMPLETED transition. The `export_status=PENDING` + `export_target`/`export_format` attributes on the order row form the outbox; the `po.export_internal` / `po.export_requested` / `po.export_error` decision records make every branch observable. The drainer — the Skill-side `export_purchase_order` transport (PRD-011 §3.2) — is the remaining Phase-3 work; until it ships, real-tenant POs accumulate as `PENDING` and the test tenant's internal PO Receiving is unaffected. Pairs with AD-9 (one PO per supplier is the unit of export) and AD-17 (durable authoritative write, best-effort downstream delivery).

The authoritative `{env}-orders` write and the outbox stamp are made atomic by the transactional-outbox `commit_with_event` (PRD-006 §2.6), a single `TransactWriteItems` dual-write, so a PO can never be authoritative without its export intent recorded (or vice-versa). PR #91 fixed the missing `dynamodb:TransactWriteItems` grant on the `step_invoker` role that Node 7 (`node_award_comms._commit_award_and_export`) needs for this commit — without it the atomic write failed `AccessDenied`.

## Amendment — the `target_system` dispatch seam (2026-09-22)

This decision's original Results left the drainer as "the remaining Phase-3 work" and named SAP's the only shape it would take. Both halves have since moved, and the way they moved is itself part of the decision.

Until impl PR #469, setting `target_system: "oracle"` did not enable Oracle export — it *broke* it. `_resolve_po_export` passed the tenant's configured value through verbatim into the `po_export` event, and the only consumer, `po_export_drain.export_to_sap`, hard-rejected anything but `sap` with a non-retryable `UnknownTargetSystem` that routed straight to the DLQ. Meanwhile Oracle's own export module had **zero call sites**. The "enabling a second ERP is a one-row config write" claim that `oracle_plan/`'s documents carried was false in the only way that matters: the row could be written, and the POs would be discarded.

The seam that fixed it keeps this ADR's shape and generalises its last hop:

- **Dispatch, not a hardcoded call.** `po_export_drain._EXPORTERS` maps a lowercased `target_system` to its exporter (`sap` → `export_to_sap`, `oracle` → `oracle_po_export.export_to_oracle`); `export_po(payload, base_urls=…)` resolves it and raises `UnknownTargetSystem` for anything unregistered. The `!= "sap"` guard moved out of the exporter and into the dispatcher, so an exporter no longer polices the routing decision it is the result of.
- **One entry contract per target: order-keyed.** Oracle's exporter matches `export_to_sap`'s signature exactly — read the `{env}-orders` row, return `"exported"` or `"already_exported"`, raise a terminal or supplier-unmapped error. The award-keyed `export_po_for_award` that existed alongside it was deleted rather than kept as a second, divergent entry point.
- **Error classes widen; metrics do not.** `_TERMINAL_ERRORS` and `_UNMAPPED_ERRORS` are tuples over both ERPs — `OracleTerminalError` subclasses `httpx.HTTPStatusError` exactly as `SapTerminalError` does — while `sap_call_errors` and `oracle_call_errors` stay separate metrics. A widened `except` is shared machinery; a blurred metric would hide which ERP is failing.
- **`UnknownTargetSystem` stays non-retryable.** It is a configuration value, not a transient failure: retrying cannot change what `target_system` says, so it goes to the DLQ for a human on the first attempt.

So the outbox's promise is unchanged — the authoritative `{env}-orders` write is never blocked by an ERP — but "which ERP" is now a registry lookup rather than an assumption baked into the only exporter. Adding a third target is a registry entry plus an exporter honouring that contract; it is still not a config write alone, and this ADR should not be read as promising that it is. Oracle's transport posture behind that seam is AD-160.


---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
