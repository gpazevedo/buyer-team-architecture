# AD-138 — One Governed Delivery Function Replaces Six Simulated Supplier-Send Paths

**Theme:** Integration, Skills, Plugins & Transports
**Catalog:** AD-138 · **Source PRD:** PRD-011 · **Status:** Accepted · **Related:** AD-97, AD-135, AD-45, AD-43, AD-22, AD-137

## Context

Every supplier-facing agent (`award_comms_llm`, `bottleneck_negotiation_llm`, `leverage_auction_llm`, `spot_bidding_llm`, `strategic_partnership_llm`) plus Node 7 (`node_award_comms.py`) had its own `_record_comm`/`_persist`/`_persist_comm` — six independent implementations that wrote a `{env}-communications` row and stopped there; nothing ever left the account. The casing of the terminal status wasn't even consistent (four tools wrote lowercase `"sent"`, `award_comms_llm`/Node 7 wrote uppercase `"SENT"`), only `award_comms_llm` read the `auto_send_communications` policy, Node 7 hardcoded `PENDING_APPROVAL`, and the other four didn't gate on the policy at all. Two of the six (`bottleneck_negotiation_llm`, `strategic_partnership_llm`) had no pre-send guard against leaking a competitor's name, price, or the buyer's own budget ceiling into a supplier-facing message. Turning simulated delivery into a real one (Amazon SES) makes every one of these gaps live and irreversible — an SES send, unlike a DynamoDB write, cannot be quietly rolled back once it leaves the account.

## Decision

Replace all six call sites with one function, `supplier_comms.delivery.deliver_communication()`, that every path funnels through:

1. **Idempotent** on `(negotiation_id, comm_type, supplier_id[, round_number])` — a re-call returns the existing row rather than double-sending. `round_number` joins the key only for the multi-round call sites (auction feedback, negotiation proposals), where the same `comm_type`/`supplier_id` pair legitimately sends more than once per negotiation.
2. **Re-validates `tenant_id`** against the `{env}-negotiations` row itself before doing anything else — never trusts an LLM-supplied `tenant_id`, even though the agent-side seam (below) never lets one reach this function as a raw tool parameter in the first place. Defense in depth, not the only control (mirrors AD-38's posture).
3. **Runs a content-safety scan** (`content_safety.scan_for_disclosure`, REQ-A200) against other candidate suppliers' names, the budget ceiling, and (for a non-winner) the awarded price, and **fails closed** — blocks the send — on any match, closing the gap the two ungated agents had.
4. **Reads `auto_send_communications` itself**, the single reader for all six call sites; off (the default) leaves the row `PENDING_APPROVAL` with zero SES calls.
5. **On auto-send**, resolves the channel (`channel.select_channel` — Supplier MCP first, EMAIL fallback; every non-EMAIL branch is a deliberate "not available in v1" terminal case, not a deleted stub, so a real Supplier-MCP transport slots in later without a signature change) and calls SES through an injected `breaker_call`.

The module never imports `pybreaker` or constructs a boto3 SES client directly — `orchestrator/comms_delivery.py` is the orchestrator-owned caller that injects the real `supplier_email` circuit breaker (a new named instance of the existing per-operation-breaker pattern, AD-45) and the only place a breaker wraps an SES send. Node 7 imports `deliver()` in-process (same shape as `po_delivery.deliver_purchase_order`, AD-97); the five agent tools reach it only through `buyer_agent_core.comms.send_supplier_communication` — a synchronous `lambda:InvokeFunction`, the same AD-135 seam boundary that already forbids `import boto3` in any agent `tools.py` — landing on `comms_delivery.lambda_handler`.

## Alternatives Considered

- **Wire each of the six call sites to SES independently.** Rejected: perpetuates the inconsistent status casing, inconsistent auto-send gating, and the two missing content-safety guards — the exact defects a single governed path exists to close. Six independent SES integrations would also mean six places to get tenant validation or fail-closed behavior wrong.
- **Content-safety as a Bedrock Guardrail (mask-and-retry) instead of a hard block.** Rejected: a Bedrock Guardrail's anonymize-and-retry posture (AD-128) is appropriate when the model can be steered to a corrected output before anything happens. An SES send is a one-shot, irreversible external side effect — masking after the fact isn't possible, so the only correct response to a scan match is to not send.
- **Skip the tenant re-validation since the agent-side seam already scopes `tenant_id`.** Rejected: the seam boundary is a code-level guarantee, not a data-level one — `deliver_communication` is also called in-process by Node 7, and a defense-in-depth check at the function itself costs one DynamoDB read against an already-fetched row.

## Trade-offs

| Gained | Given up |
| --- | --- |
| One place to get idempotency, tenant scoping, content-safety, and the auto-send gate right, instead of six | A synchronous Lambda-invoke hop (agent tools → `comms-delivery`) on every send, versus the in-process call the old simulated paths made |
| Fail-closed content safety on every supplier-facing send, including the two agents that previously had no guard at all | A scan match now blocks a send outright rather than degrading gracefully — a false positive costs a manual resend, not silent delivery |
| A circuit breaker on the one real external side effect this system has ever made (previous integrations are all internal AWS services) | A new single point of failure: `comms-delivery`'s own availability now gates every supplier communication, where six independent paths had none |

## Results

Realized across `packages/supplier_comms/` (`delivery.py`, `channel.py`, `content_safety.py`, `models.py`, `ses_transport.py`), `orchestrator/comms_delivery.py`, and `packages/buyer_agent_core/buyer_agent_core/comms.py` (impl PRs #270–#279, #285, 2026-08-12/13 — Steps 0 through 6 of the migration). All six call sites (`award_comms_llm`, `bottleneck_negotiation_llm`, `leverage_auction_llm`, `spot_bidding_llm`, `strategic_partnership_llm`, Node 7) are wired to the real seam and deployed to dev.

Step 2 (impl PR #272) had to decouple the Lambda interface VPC endpoint the five agent runtimes need to reach `comms-delivery` from dev's existing `enable_interface_endpoints` flag, which was tied 1:1 to the NAT-pause toggle (AD-133's pause mechanism) — reusing it would have torn the endpoint down on the next unrelated pause. A dedicated `enable_lambda_endpoint` variable gates it instead, fixed `true` independent of NAT state.

`auto_send_communications` remains off by default in dev, so every row observed so far is `PENDING_APPROVAL` — the SES send path, the circuit breaker, and the content-safety block-path are exercised in tests (`packages/supplier_comms/tests/`, `orchestrator/tests/test_comms_delivery.py`) but not yet by a live auto-sent email. `select_channel` resolves to EMAIL unconditionally today — no real Supplier-MCP transport exists to select instead.
