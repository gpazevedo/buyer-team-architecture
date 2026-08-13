# AD-141 — REQ-S607 Descoped: Contextual Grounding Check Has No Target (REQ-S607)

**Theme:** Security, Governance & Trust Boundaries
**Catalog:** AD-141 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-117, AD-43

## Context

PRD-005's REQ-S607 specified a Bedrock Guardrails contextual grounding check (grounding/relevance thresholds 0.75) on Bid Evaluation Agent responses, part of the v1.1.0 ATLAS uplift's Defense Evasion coverage. AD-117 (2026-07-06, `buyer-team-impl` PR #151) removed Bid Evaluation as an LLM agent: Node 5 is deterministic Python (`orchestrator/scoring.py` / `node_bid_evaluation.py`) running under the Step Functions Orchestrator IAM role, not a Gateway-fronted Bedrock call. AD-43 — the ADR that scoped the live Guardrail's actual policy content — never included this check either; it was specified in PRD-005 but never built.

REQ-S607's target and its would-be enforcement point are now the same fact stated twice: there is no Bedrock invocation anywhere in the Bid Evaluation path for a contextual grounding check to attach to.

## Decision

Formally descope REQ-S607 rather than build it. Building a contextual grounding check would require re-introducing a Bedrock call into Node 5 specifically to give the check something to run against — reversing AD-117's deliberate removal for no reason connected to that decision's own rationale (Node 5's scoring logic is deterministic and auditable; a Bedrock round-trip was architectural overhead, not a capability gap). PRD-005 §10.7's REQ-S607 entry is marked `SUPERSEDED, non-normative`, following the convention PRD-006 §7.4 already established for requirements whose implementing mechanism was deliberately deleted (REQ-R300–R302).

## Alternatives Considered

- **Build it against some other node's Bedrock call** (e.g. attach grounding checks to Award & Comms or Strategic Negotiation instead). Rejected: REQ-S607 is specifically about Bid Evaluation output; retargeting it to a different agent would be inventing a new requirement under an old ID, not implementing this one.
- **Reintroduce a lightweight Bedrock call into Node 5 purely to host the check.** Rejected: this is a strictly worse system than today's — it adds latency, cost, and a new LLM failure mode to a node AD-117 made deterministic on purpose, purely to satisfy a spec line rather than close a real gap.
- **Leave REQ-S607 marked "not implemented" indefinitely.** Rejected: "not implemented" implies future work is expected; that is false here. `SUPERSEDED, non-normative` is the accurate status — no future PR will make this true as written.

## Trade-offs

| Gained | Given up |
| --- | --- |
| PRD-005's status table stops carrying a permanently-open item that reads as a gap to any future reconciliation pass, closing the loop AD-117 left dangling | The Defense Evasion ATLAS row loses this specific control; coverage there continues to rely on the cross-negotiation supplier-skew detector and online quality composite (PRD-004 §4.3/§4.5), already documented as Partial |
| Matches the PRD-006 §7.4 precedent for requirements whose implementing mechanism was deleted as dead code — one convention across the PRD suite instead of a bespoke "not implemented" note that never resolves | If Bid Evaluation is ever reintroduced as an LLM agent, REQ-S607 would need to be revived from this superseded state rather than picked up as still-open |

## Results

PRD-005 §10.7's REQ-S607 entry retagged `SUPERSEDED, non-normative`, rationale pointing at AD-117 and this ADR (spec v1.9.4). No code or infrastructure change — this ADR only closes the requirement's status, matching the PRD-006 §7.4 convention.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
