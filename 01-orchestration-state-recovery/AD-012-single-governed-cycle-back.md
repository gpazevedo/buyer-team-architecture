# AD-012 — Single Governed Cycle-Back (Node 6 → Node 4x, Max 1)

**Theme:** Orchestration, State & Recovery
**Catalog:** AD-12 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-11, AD-14, AD-16

## Context

A human approver at Node 6 may want a negotiation re-run rather than awarded or cancelled. Modelling this as a flag on *reject* conflates two different intents — "this award is wrong, cancel it" versus "try again". Allowing re-negotiation also introduces the classic risk of unbounded loops: cost blowout, indefinite stalling, and stale bid data from the rejected round contaminating the next evaluation.

## Decision

Model the Node 6 human decision as three first-class outcomes — `APPROVED`, `REJECTED`, `CYCLE_BACK` — not a reject with a retry flag. `REJECTED` is always terminal (→ CANCELLED); `CYCLE_BACK` is the distinct "send back for re-negotiation" decision. Permit exactly one `CYCLE_BACK` from Node 6 to the originating Node 4x (REQ-G204, max 1); a second `CYCLE_BACK` is exhausted → `REQUIRES_ATTENTION` (`cycle_back_exhausted`, AD-016 trigger #18). The re-run routes back through the Strategy Router, so Node 6 echoes `kraljic_quadrant` on the cycle-back output for the router to re-branch on. Before the strategy agent is reinvoked, all `Bid` entities for the negotiation are set to `SUPERSEDED`, and Node 5 filters `status != SUPERSEDED` so the rejected round can never re-enter evaluation. The SUPERSEDED write is retried up to three times; if it fails, the cycle-back is *halted* and the negotiation goes to `REQUIRES_ATTENTION` (`superseded_marking_failure`) rather than proceeding on unconfirmed state.

## Alternatives Considered

- **Cycle-back as a `reject`-with-retry flag.** Rejected: overloads "reject" with two intents (cancel vs re-run), so callers and audit trails can't tell them apart; a distinct `CYCLE_BACK` decision keeps `REJECTED` unambiguously terminal.
- **Unbounded retries.** Rejected: infinite-loop and cost risk; no natural termination.
- **No cycle-back at all (only approve/reject).** Rejected: forces a full new negotiation for what may be a single-round correction, losing context.
- **Cycle-back without bid cleanup.** Rejected: stale bids from the rejected round would leak into Node 5 and corrupt the recommendation.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Worst-case negotiation cost and duration are bounded and predictable | A single retry may still land on a suboptimal award if one round is insufficient |
| Node 5 is guaranteed a clean bid set; re-marking already-SUPERSEDED bids is a no-op on recovery | SUPERSEDED cleanup adds a new failure surface that must be handled before proceeding |

The cleanup failure is deliberately resolved by halting to REQUIRES_ATTENTION rather than proceeding — a safe stall is preferred over a possibly-corrupt evaluation.

## Results

Worst-case negotiation cost and duration are predictable (at most one extra strategy round). Node 5 is guaranteed a clean bid set. The re-marking is idempotent, so cycle-back recovery re-running the SUPERSEDED step is a no-op. Note that for NON_CRITICAL spot bids below `auto_award_below_threshold_usd`, cycle-back is unreachable because they auto-approve without human review. All three Node 6 decisions (`APPROVED`/`REJECTED`/`CYCLE_BACK`) are live-validated end-to-end against real Step Functions on dev — the CYCLE_BACK path proves the loopback re-runs the strategy and re-pauses at the gate (fails closed if the `kraljic_quadrant` echo is dropped, since the Strategy Router would otherwise route to the terminal default). See AD-016 for the `cycle_back_exhausted` trigger.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
