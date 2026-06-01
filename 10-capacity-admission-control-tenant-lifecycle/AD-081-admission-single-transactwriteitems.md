# AD-081 — Admission Is a Single TransactWriteItems with an ACTIVE ConditionCheck

**Theme:** Capacity, Admission Control & Tenant Lifecycle  **Catalog:** AD-81 · **Source PRD:** PRD-016 · **Status:** Accepted · **Related:** AD-80, AD-82, AD-85

## Context

The admission decision (increment counters) and the lifecycle write (`DRAFT → ACTIVE`) must not be able to diverge — a counter incremented without the corresponding status write, or vice versa, leaves orphaned capacity that permanently shrinks a tenant's effective slots. Admission must also be refused for tenants that are not operationally live (PENDING, SUSPENDED, DECOMMISSIONED), and that check must be atomic with the counter increment so a status change mid-admission cannot produce a race.

## Decision

Execute the entire admission as one DynamoDB `TransactWriteItems`: an item-0 `ConditionCheck` requiring the tenant row's `status = ACTIVE`, plus conditional counter increments (per-tenant and global), and the negotiation `DRAFT → ACTIVE` status write. Either all succeed atomically or none do. Counter decrements fire on terminal transitions (`AWARDED`, `CANCELLED`) in the same transaction as the status write. `REQUIRES_ATTENTION` deliberately does not decrement — it holds the slot, since it can resume to `ACTIVE` without re-admission.

## Alternatives Considered

- **Separate read-then-write for the status check.** Rejected: a non-atomic status check leaves a TOCTOU window where a tenant is suspended between the check and the counter increment.
- **Optimistic locking with a conditional update and separate status check.** Rejected: two round-trips cannot be made atomic; a crash between them produces orphan counters or ghost admissions.
- **Include the budget check in the transaction.** Rejected: a budget-vs-config comparison cannot be expressed as a DynamoDB transaction condition — it requires reading an aggregated counter and comparing it to a config value, which forces the spend gate out of the transaction (AD-82).

## Trade-offs

| Gained | Given up |
| --- | --- |
| Atomicity eliminates orphan-counter races on the entire admission path | Everything in the admission decision must be expressible as DynamoDB transaction conditions; the budget check is not (forces AD-82) |
| PENDING/SUSPENDED/DECOMMISSIONED tenants are rejected before capacity is even consulted | `TransactWriteItems` has item and throughput limits; the `_global` counter is a single-partition hot spot (sharding deferred, out of v1.0 scope) |
| Rejection reason is derived from the tenant's actual status, not a generic quota error | |

## Results

Runs at Node 1 between validation and the negotiation status transition; the Orchestrator is the only component that sees every negotiation. Decrements fire on terminal transitions in the same transaction as the terminal status write. `REQUIRES_ATTENTION` holds its slot (AD-16). The spend gate (AD-82) runs as a non-atomic pre-check before this transaction. The tenant status conditions read exactly the states defined by AD-85.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
