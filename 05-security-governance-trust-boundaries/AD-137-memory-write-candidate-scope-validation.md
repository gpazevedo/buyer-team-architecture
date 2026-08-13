# AD-137 — Memory-Write Authorization via Candidate-Scope Validation, Not a JWT Claim Check (REQ-S608)

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-137 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-129, AD-136, AD-135

## Context

PRD-005's REQ-S608 (ATLAS Persistence — Memory Manipulation) specified authenticating memory writes by validating a workload-identity JWT's `tenant_id` claim at the write boundary, via a proposed `agents/shared/memory_auth.py`. That module was never buildable as specified: the only real memory write is AD-129's `supplier-memory` DynamoDB table, written from `orchestrator/node_strategy_execute.py:_record_supplier_memory` — a Step Functions Lambda node, not an agent runtime — so no workload-identity token exists anywhere in that call path to validate. `StrategicPartnershipResponse` also carries no `tenant_id` field; `tenant_id` reaches the write from the orchestrator's own trusted Step Functions event, never from agent output, so a claim check would validate a value against itself.

Tracing the actual write path (rather than building the spec'd control verbatim) surfaced a real vulnerability instead: `_record_supplier_memory` iterated `result["tco_analysis"]` — the strategic-partnership agent's own LLM output — and wrote one cross-negotiation history row per agent-supplied `supplier_id`. `_query_supplier_memory` reads those rows back as relationship history the next time that supplier is a candidate. A prompt-injected agent (ATLAS Persistence — Memory Manipulation) could therefore seed history for suppliers that were never in the negotiation, poisoning every later negotiation involving them. The sibling function `_apply_negotiated_terms` was never exposed to this class of bug: it iterates the trusted candidate set (`sup_to_bid`) and looks the agent's output up *by key*, so a fabricated supplier id is silently ignored there. The asymmetry between the two functions — one trusts the candidate set, the other trusts the agent — was the actual defect.

## Decision

Authorize memory writes against the orchestrator's own trusted candidate set, not an identity claim. `_record_supplier_memory` now takes `candidate_supplier_ids: set[str]` (`set(sup_to_bid)`, seeded by the orchestrator from the durable negotiation row — never anything the agent returned) plus `agent_name`. Any agent-supplied `supplier_id` outside that set, or missing entirely, is refused before the DynamoDB write via a new `_reject_memory_write`, which emits `security.memory_write.rejected` to `procurement/resilience` (EMF, dimensions `{agent_name, reason}`; reasons `supplier_not_a_candidate`, `missing_supplier_id`) and logs the rejected identifiers on the structured `_decision` record rather than as metric dimensions, keeping cardinality fixed regardless of how many distinct suppliers get named. A new CloudWatch alarm, `memory_write_rejected`, fires at ≥1 rejection in 5 minutes to the same `evaluation_alerts` SNS topic AD-136 uses. This makes the memory-write path match the pricing path's existing trust boundary instead of adding a new authentication mechanism the architecture has no caller for.

## Alternatives Considered

- **Build the spec'd JWT `tenant_id`-claim check verbatim.** Rejected: no workload-identity token exists in a Step Functions Lambda's call path, and `tenant_id` was never agent-controlled here — the module would have had no caller and no attacker-controlled value to check, i.e. dead code satisfying the letter of REQ-S608 while leaving the real vulnerability (agent-controlled `supplier_id`) open.
- **Do nothing, since the write path is "only" reachable from a trusted orchestrator Lambda.** Rejected once traced: the orchestrator is trusted, but the *content* of `result["tco_analysis"]` it writes is the strategic-partnership agent's own LLM output, which is exactly the untrusted surface ATLAS Persistence — Memory Manipulation describes. Trusting the caller is not the same as trusting the caller's inputs.
- **Reject at read time (`_query_supplier_memory`) instead of write time.** Rejected: poisoned rows would still accumulate in the table for every negotiation between the write and the (hypothetical) read-time filter, and a read-time filter can't emit a clean rejection signal tied to the agent that fabricated the id.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Closes a real, demonstrated memory-poisoning vector (agent-fabricated `supplier_id` persisted and read back into later negotiations) rather than shipping a claim check with no attack surface to close | The spec's JWT `tenant_id`-claim design stays unbuilt; REQ-S608 is satisfied by re-scoping the requirement to the real threat, not by building it as literally specified |
| Matches the trust model `_apply_negotiated_terms` already used for pricing — one mental model for "agent output must be looked up against the candidate set, never trusted directly" | The JWT-claim form remains a real gap *if* AgentCore Memory or Mem0 is ever wired (AD-129), or if any agent gains a direct memory-write tool — neither is true today, but this decision doesn't extend to that future shape |
| Fixed-cardinality metric (`{agent_name, reason}`) keeps the alarm cheap regardless of supplier-name churn, consistent with the cardinality discipline elsewhere in `bucket_a_alarms.tf` | Rejected-write detail (which supplier, which negotiation) lives only in structured logs, not the metric — an operator paged by the alarm must go to logs for the specific incident |

## Results

Shipped and applied to dev: `orchestrator/node_strategy_execute.py`'s `_record_supplier_memory` + new `_reject_memory_write` (impl PR #264, `7230bdc`, 2026-08-06); `aws_cloudwatch_metric_alarm.memory_write_rejected` in `infra/bucket_a_alarms.tf`, confirmed present in `terraform state list` after a same-day dev image re-deploy (`69c63c8`, "re-fire dev deploy for 7230bdc") rebuilt the orchestrator Lambda via `scripts/build_orchestrator_lambda.sh`. PRD-005 §1.3's Memory Manipulation row is updated None → Covered and §10.7's REQ-S608 entry rewritten with As built / vulnerability / deviation subsections (spec v1.9.3, impl v1.10.3). AD-129 is updated with a forward pointer here, since this decision adds an authorization layer on top of the table AD-129 introduced without one.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
