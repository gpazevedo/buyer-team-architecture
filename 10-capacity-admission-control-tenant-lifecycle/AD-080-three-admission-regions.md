# AD-080 — Three Admission Regions; Below-Reservation Skips the Global Check

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-80 · **Source PRD:** PRD-016 · **Status:** Accepted · **Related:** AD-79, AD-81

## Context

Given the reserved/max/weight model of AD-79, an incoming negotiation request always falls into one of three situations relative to the tenant's own usage and the global shared pool. Each situation warrants different admission behavior: a tenant within its reservation should never contend with other tenants' bursts, while a tenant attempting burst access must compete fairly for available shared capacity.

## Decision

Define three admission regions based on a tenant's current `in_flight` counter: **below reservation** (`in_flight < reserved_slots`) — admit unconditionally and skip the global counter check; **in burst** (`reserved_slots ≤ in_flight < max_slots`) — admit only if `global_in_flight < G − Σ_other_tenants reserved_slots`; **at ceiling** (`in_flight ≥ max_slots`) — reject with backpressure. Rejection reasons differ by region: `tenant_quota_exhausted` at ceiling, `shared_pool_exhausted` in burst with no shared capacity available.

## Alternatives Considered

- **Single global ceiling with no per-tenant regions.** Rejected: a single counter cannot distinguish reserved from burst usage and cannot provide the guarantee that a below-reservation tenant never contends with bursting tenants.
- **Two regions (below-max vs at-ceiling) without a reservation fast path.** Rejected: all below-max requests would hit the global counter, creating cross-tenant contention latency even for tenants using only their reserved capacity.
- **Status quo / no differentiated regions.** Rejected: without differentiation, burst traffic from one tenant can exhaust global capacity and block another tenant's reserved slots.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Below-reservation requests are latency-free of cross-tenant contention | Burst admission reads a global counter, adding cross-tenant contention latency on the burst path |
| `Σ_other reserved` subtraction in the burst check protects every other tenant's guaranteed floor | Three-region logic plus the conditional skip is more complex than a single ceiling check |
| Distinct rejection reasons and `Retry-After` semantics per region aid client backoff | Below-reservation fast path requires reading the tenant counter first before deciding to skip the global check |

## Results

Encoded directly in the admission transaction (AD-81): the global-counter condition expression is omitted when `current_in_flight + 1 ≤ reserved_slots`. `Retry-After` values differ by rejection reason. The burst region's global condition is what prevents one tenant's burst from consuming capacity reserved for others.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
