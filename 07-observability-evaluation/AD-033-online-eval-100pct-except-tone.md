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

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
