# AD-091 — `max_concurrent_bids = 200` (Reject the Proposed 200→20 Reduction)

**Theme:** Capacity, Admission Control & Tenant Lifecycle
**Catalog:** AD-91 · **Source PRD:** PRD-016, PRD-006 · **Status:** Accepted · **Related:** AD-50, AD-79

## Context

In early capacity planning a request was raised to reduce `max_concurrent_bids` from 200 to 20 to limit cost exposure from the Spot Bidding Agent's bid fan-out. The concern was that 200 concurrent bids across all auctions could, in the worst case, produce a large billable token volume in a short window.

Two different knobs govern bid concurrency, and they had been conflated:

- **`max_concurrent_bids`** — the global throughput pool (200). The maximum number of in-flight `send_bid_invitation` tool invocations across *all* auctions simultaneously.
- **`spot_bid_invitations_per_auction`** — the per-auction invitation cap (20). The maximum number of suppliers any single negotiation can invite.

Per-negotiation token cost scales with invited-supplier count N, not concurrency. The cost-relevant cap is therefore already 20 (the per-auction cap). Dropping the global pool from 200 to 20 would throttle throughput without reducing per-negotiation cost.

## Decision

Keep `max_concurrent_bids = 200`. Reject the proposed reduction to 20.

The 200-slot global Spot Bid bulkhead is sized to the Bedrock `InvokeAgentRuntime` 25 TPS per-agent limit and the <90s bid-fan-out SLA (PRD-003 §2.2). AgentCore session-per-microVM means the bulkhead *is* the throttle — there is no instance pool to scale.

20 would bottleneck at 13 concurrent negotiations (1 supplier each), while 200 supports 10 concurrent 20-supplier auctions. The per-auction cap (`spot_bid_invitations_per_auction = 20`) already bounds per-negotiation cost; the global pool exists to bound aggregate throughput against the API rate limit, not cost.

## Alternatives Considered

- **Reduce `max_concurrent_bids` to 20.** Rejected: (1) The cost lever is the per-auction cap, not the global pool — per-negotiation token cost scales with invited-supplier count N (`spot_bid_invitations_per_auction = 20`), not with the number of concurrent auctions. (2) Dropping to 20 would invalidate the 200-in-90s bid-fan-out SLA (PRD-003 §2.2 / PRD-001 §8.2), the dataset-3 load framing (PRD-001 §6.2), the PRD-006 §2.4 bulkhead, and the PRD-016 §2 capacity math. (3) The cost-containment concern is already addressed by the existing per-auction cap.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Sustains the 200-bid-in-90s fan-out SLA for auctions with up to 20 suppliers | None — the cost-containment concern that motivated the reduction is already met by the per-auction cap |

The decision is recorded primarily to close the open question, not because the trade-off was close. All citing sites already used 200 as the canonical value; no sweep was required.

## Results

`max_concurrent_bids = 200` confirmed in PRD-001 §5.4.1, PRD-003 §2.2, PRD-006 §2.4, PRD-016 §2/§8, and the cross-PRD invariants table. The two-knob distinction (`max_concurrent_bids` vs `spot_bid_invitations_per_auction`) is now explicitly documented across all citing sites.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
