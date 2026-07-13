# AD-033 — Online Evaluation on 100% of Traces, Except Communication Tone (Tiered Sampling)

**Theme:** Observability & Evaluation  **Catalog:** AD-33 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-32, AD-34

## Context

Online evaluation runs against 100% of production traces. Communication Tone is the one evaluator that runs per supplier message rather than once per negotiation, making it the primary evaluation cost driver. Most supplier messages are routine and low-stakes; evaluating every one at full LLM-as-Judge cost would dominate the evaluation bill without proportionate quality benefit. At the same time, award and rejection notifications are legally binding and relationship-sensitive, and high-value negotiations warrant heightened scrutiny.

## Decision

Keep 100% online coverage for all evaluators except Communication Tone, which follows a tiered sampling strategy:

- **100% coverage** for award and rejection notifications (legally binding; relationship- and confidentiality-sensitive), and for any communication on a negotiation whose total value exceeds `communication_tone_full_coverage_threshold_usd` (default $50,000).
- **Sampled coverage** at `communication_tone_sampling_rate` (default 10%) for all other communications below the threshold.

Both flags support per-tenant DynamoDB overrides via the two-stage threshold resolution (PRD-010 §3.1).

## Alternatives Considered

- **Flat 100% Communication Tone coverage.** Rejected: Communication Tone's per-message cardinality makes full coverage the dominant evaluation cost with no proportionate benefit on routine low-value messages.
- **Flat sampled coverage for all communication types including award/rejection.** Rejected: award and rejection notifications are legally binding and confidentiality-sensitive; sampling them creates an unacceptable blind spot on the highest-risk communications.

## Trade-offs

| Gained | Given up |
| --- | --- |
| The single largest evaluation cost is cut sharply by sampling the routine majority of low-value messages | A tone regression confined to sampled-out messages may go undetected until it happens to be sampled |
| Full scrutiny is preserved on the highest-risk communications — award/rejection and high-value deals | Per-tenant overrides add configuration surface that must be reasoned about per customer |
| Scrutiny is tunable per tenant for customers with stricter requirements | The residual exposure on low-value routine messages is an explicitly accepted blind spot |

## Results

The single largest evaluation cost is cut sharply while full scrutiny is preserved on the communications where tone matters most. The sampled fraction is tunable per tenant for customers with stricter tone requirements. The residual exposure — incomplete tone coverage on low-value routine messages — is an explicitly accepted blind spot. AD-34 (closed-loop automated actions) consumes the resulting Communication Tone scores, including the alarm that disables auto-send when Tone drops below 0.80.

**Not yet built (as of PR #108, 2026-07-02).** This sampling strategy presupposes an online Communication Tone evaluator wired into the per-trace path; per AD-32's Results, no such evaluator has been observed deployed — only an offline LLM-judge stub exists (`evals/llm_judge.py`, not this tone evaluator specifically). The tiered-sampling mechanics described here remain the intended design but are unimplemented; there is no live coverage, sampled or otherwise, to report.

**Update 2026-07-02 (deferred-scope build-out, Phase 4): sampling is now implemented, at `orchestrator/node_award_comms.py`.** Every persisted award notification is scored at 100%; rejection notifications are scored at 100% when the negotiation's `awarded_price` is at/over `communication_tone_full_coverage_threshold_usd` (default $50,000, from the governance config's `evaluation` block), else sampled at `communication_tone_sampling_rate` (default 10%) via a deterministic hash of the communication id (stable across a Step Functions retry — not `random.random()`). Both thresholds resolve through the same two-stage governance-config precedence `load_governance_config` already uses (profile + tenant overlay), not a separate config path. The judge itself was Phoenix-backed at the time of this update (`evals/phoenix_judge.py::score_communication_tone`, see AD-032 Results) and live-verified against Bedrock; **PR #127 (2026-07-03, one day later) replaced it with `evals/tone_negotiation_judge.py::score_communication_tone`** (raw Bedrock `converse()`, `phoenix_judge.py` deleted — see AD-032's correction note). Same scoring/thresholds, different judge module. **Scope limit: this gate only covers award/rejection notifications (Node 7's domain).** Negotiation-round communications from the bidding/negotiation agents (bottleneck, leverage-auction, strategic-partnership) are not wired to any sampling gate or Communication Tone scoring — this ADR's "for any communication on a negotiation whose total value exceeds the threshold" clause is unimplemented for that communication type. Still no AD-34 alarm (the 0.80 auto-send-disable threshold this Results section originally described is not wired — metric-only, per this build-out's explicit scope). (The alarm itself was provisioned two days later, 2026-07-04 — see AD-34's Results — though the automated disable-auto-send action still isn't.)

**Update 2026-07-12 (PR #202/#203): Communication Tone scoring moved off the synchronous award/PO-issuance path.** `node_award_comms.py::award_comms()` now issues the PO and commits its compensations first, then fires an asynchronous self-invoke of its own Lambda (`InvocationType="Event"`, payload `{"_eval_only": True, tenant_id, negotiation_id, award_id}`) that runs both Communication Tone and the confidentiality check out of process (`_run_eval_only`, PR #202). This removed roughly 10.8s of synchronous Bedrock judge latency from every award. PR #203 (same day) bounded the self-invoke call itself with a tight `connect_timeout=2`/`read_timeout=3` and zero retries, so a slow/throttled `lambda:Invoke` fails fast into the existing `except Exception` handler instead of blocking the already-committed award. The tiered-sampling decision this ADR describes (100% award/rejection, sampled otherwise) is unchanged by this move — only *when* the score is computed changed, not *which* communications get scored. Accepted trade-off: a judge failure on the async path (Bedrock throttling, timeout) now means that score is simply lost — no retry, no re-queue — since the PO is already issued and there's nothing left to gate. Negotiation Quality is not part of this self-invoke; per AD-32's 2026-07-06 correction, it hasn't been scored live since PR #151, independent of this change.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
