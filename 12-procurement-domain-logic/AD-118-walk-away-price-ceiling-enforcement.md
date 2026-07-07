# AD-118 — Walk-Away Price Ceiling: Orchestrator Enforces Budget on Agent-Returned Prices

**Theme:** Procurement Domain Logic
**Catalog:** AD-118 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-023, AD-018

## Context

The four strategy agents (Spot Bidding, Leverage Auction, Bottleneck Negotiation, Strategic Partnership) return priced bids whose amounts the orchestrator writes directly into `{env}-bids`. The `budget_limit` field on each PR item represents the walk-away price — the maximum the buyer is willing to pay. Agents are instructed via system prompt to never exceed this limit ("reserve — never reveal to suppliers"), but a prompt-only convention is not an invariant: prompt drift, model updates, or adversarially crafted inputs can cause an agent to return an above-ceiling amount, and nothing in the code path blocked it.

The existing graceful-degradation path already handles agent invocation failures by pricing bids deterministically at a fallback amount. The same path should fire when the agent returns a valid response whose price exceeds the ceiling — the response is structurally valid but violates a hard business invariant.

## Decision

Add `_over_ceiling(amount, budget_limit)` as a shared helper. In each of the three write-back functions that apply agent prices to bid rows, check every agent-returned amount against the item's `budget_limit` before writing. Above-ceiling bids are rejected and priced at the deterministic fallback, with a source label indicating which strategy's ceiling was violated (`spot_bidding_ceiling_clamp`, `leverage_ceiling_clamp`, `bottleneck_negotiation_ceiling_clamp`, `strategic_partnership_ceiling_clamp`). The fallback amount is derived from a deterministic formula keyed off the bid ID, same as the agent-failure fallback path.

This turns "reserve — never reveal to suppliers" from a prompt convention into a code-enforced invariant. The agent still owns the pricing decision within the [0, budget_limit] range; the orchestrator owns the ceiling.

## Alternatives Considered

- **Reject the entire negotiation when any bid exceeds the ceiling.** Rejected: too aggressive — a single over-ceiling bid from one supplier shouldn't cancel the negotiation when other suppliers priced within budget. The per-bid clamp preserves the valid bids.
- **Return the over-ceiling amount to the agent with a correction request.** Rejected: adds another LLM round-trip (latency + cost) for what is a deterministic correction. The fallback path already computes a valid price.
- **Silently cap at budget_limit.** Rejected: a capped amount would misrepresent the supplier's actual bid to downstream steps (award selection, savings calculation). The fallback amount with a distinct source label makes the override observable and auditable.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Budget ceiling enforced invariantly across all four strategies — no prompt drift can expose the ceiling | An agent that consistently returns above-ceiling amounts wastes LLM invocations (the fallback fires per over-ceiling bid) — but this is self-limiting since the agent's own structured output schema constrains amounts to numbers |
| Reuses the existing fallback path — no new code path, no new failure mode | The fallback amount is deterministic and not market-informed; a supplier whose agent consistently overprices gets the same hash-derived price every time |

## Results

Shipped in PR #143 (merged 2026-07-05). Added `_over_ceiling` helper and ceiling checks in `_apply_priced_bids` (spot), `_apply_auction_bids` (leverage), and `_apply_negotiated_terms` (bottleneck + strategic). 6 new tests: above-ceiling + at-ceiling per write-back function. 172/172 tests pass.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
