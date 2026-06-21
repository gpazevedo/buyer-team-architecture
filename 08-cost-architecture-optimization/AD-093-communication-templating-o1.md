# AD-093 — Communication Generation Is O(1) in Supplier Count

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-93 · **Source PRD:** PRD-009, PRD-003 · **Status:** Accepted · **Related:** AD-57, AD-59, AD-22

## Context

Spot Bidding (Node 4a) and Award & Communications (Node 7) each generated one LLM communication per supplier. For Spot Bidding, the bid invitation body was composed per-supplier — N LLM calls for N suppliers. For Award & Communications, each rejection notification was individually LLM-generated, plus the winner notification.

The cost analysis in PRD-009 §2.1 showed that at N=10 suppliers, these per-supplier LLM calls were the dominant cost driver, consuming ~$0.009 worth of tokens per negotiation. More importantly, the N-per-supplier pattern meant:
- The `BidConfidentialityGuard` steering hook had to run N times (once per invitation generation), multiplying the security surface that needs auditing.
- Per-supplier LLM invitation generation introduced a latent fairness gap: the model could word the same RFQ differently for different bidders.

## Decision

Collapse both communication paths from O(N) to O(1):

- **Spot Bidding:** The agent makes one `compose_invitation_body` LLM call per negotiation to produce the canonical RFQ body. Each supplier's invitation is then a deterministic render of the shared body with the addressee field filled — no per-supplier LLM call. `compose_invitation_body` is idempotent per `(negotiation_id, type=INVITATION_BODY)`.
- **Award & Comms:** Rejection notifications are rendered from a governed template (deterministic, zero LLM tokens). Only the winner award notification uses an LLM call. The `WinnerDisclosureGuard` steering hook switches from MODIFY (redacting free text) to REJECT (asserting a bounded field set), since the template parameter set has no winner-identifying fields by construction.

Per-supplier LLM work that legitimately remains O(N): reading each bid response and composing targeted clarifications for incomplete bids (Spot Bidding, REQ-A202).

## Alternatives Considered

- **Keep per-supplier generation.** Rejected: unnecessary cost and the latent fairness gap where per-vendor LLM generation could word the same RFQ differently. The content is identical for all suppliers in both cases (invitations share the same RFQ, rejections share the same "you did not win" message), so per-supplier LLM generation is wasteful.
- **Template-only generation for everything.** Rejected: `compose_invitation_body` benefits from LLM flexibility to adapt item descriptions, delivery context, and terms to the specific negotiation. The winner notification similarly benefits from LLM-generated rationale. The O(1) pattern applies templates only where the output content is invariant (rejections) or the input is shared (RFQ body).

## Trade-offs

| Gained | Given up |
| --- | --- |
| ~$0.009 saved per negotiation at N=10 | `compose_invitation_body` must be idempotent to survive crash-and-recovery without regenerating |
| Elimination of per-vendor fairness gap in RFQ wording | The `BidConfidentialityGuard` must sanitise the shared body once rather than per-supplier — simpler surface, but the single body is now the only generation to audit |
| `WinnerDisclosureGuard` changes from MODIFY to REJECT on a bounded field set — simpler enforcement, stronger guarantee | |

## Results

Implemented in PRD-003 §2.2 (Spot Bidding invitation broadcast as single LLM call + deterministic render) and PRD-003 §2.7 (Award & Comms rejection-notification templates, winner-only LLM). Cost model updated in PRD-009 §2.1–2.2 (the prior ~$0.086 figure superseded by ~$0.035 cached / ~$0.043 on-demand). Per-supplier invitation SLA revised to remove the LLM generation budget.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
