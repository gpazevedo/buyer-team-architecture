# AD-035 — ATLAS-Specific Evaluators Run on a Schedule

**Theme:** Observability & Evaluation  **Catalog:** AD-35 · **Source PRD:** PRD-004 · **Status:** Accepted · **Related:** AD-34, AD-32, AD-43

## Context

ML-integrity and adversarial threats from the MITRE ATLAS model — prompt injection, context poisoning, goal hijacking, and systematic supplier favoritism — are not caught by the per-trace quality and tone evaluators. Some of these threats (cross-negotiation supplier skew) are statistical patterns that only emerge across many negotiations over time, not within a single trace. Adversarial robustness requires replaying a curated dataset against live endpoints, which is not feasible per trace. Neither threat class fits the online per-trace evaluation model.

## Decision

Add two purpose-built evaluators that run on a schedule rather than per trace:

- **Adversarial Robustness Evaluator** (Lambda, ATLAS: Defense Evasion / ML Attack Staging): replays a curated ATLAS adversarial-prompt dataset (prompt injection variants, context poisoning, goal hijacking) against agent endpoints, verifying that Bedrock Guardrails blocks each attack and steering hooks fire. Score = blocked / total; target 1.0. Cadence: weekly in staging, monthly in production shadow. A score below 1.0 blocks the next deployment stage (REQ-O215) and emits `security.adversarial_eval.failure`. Dataset maintained in `evals/datasets/atlas-adversarial/` and updated after each red-team exercise (PRD-008).
- **Cross-Negotiation Supplier Skew Detector** (Lambda, ATLAS: ML Integrity Erosion / AML.T0031): over a rolling 90-day window per tenant, compares each supplier's observed top-ranked frequency against the frequency expected from bid scores, flagging any supplier where observed/expected ratio exceeds 2σ above the mean. Runs weekly; each run is tenant-isolated. Findings are published to `procurement/security` as `security.supplier_skew.detected` and routed for manual compliance review within 5 business days.

## Alternatives Considered

- **Inline per-trace adversarial checks.** Rejected: replaying a full adversarial dataset per production trace is cost-prohibitive and latency-incompatible with the online evaluation path.
- **Manual red-team exercises only (no automated scheduled evaluator).** Rejected: manual exercises are infrequent and do not provide a continuous, deploy-gating signal; adversarial regressions would go undetected between exercises.
- **Status quo / relying on per-trace evaluators for ATLAS threats.** Rejected: per-trace evaluators cannot detect statistical skew patterns that only emerge across many negotiations, nor can they replay adversarial datasets against endpoints.

## Trade-offs

| Gained | Given up |
| --- | --- |
| ML-integrity attacks invisible to per-trace evaluators are detected, and adversarial regressions gate releases before they reach production | Detection latency of up to one week (skew, adversarial-staging) or one month (adversarial production shadow) |
| Supplier-favoring manipulation is surfaced for compliance action with a defined 5-business-day SLA | The adversarial dataset must be refreshed after every red-team exercise; a stale dataset quietly weakens the guarantee |
| Adversarial failures block deployment, treating security regressions like build failures | Coupling release velocity to evaluator health means a broken evaluator infra can block otherwise-valid deploys |

## Results

ATLAS-class threats that the per-trace quality evaluators cannot see are detected on a defined cadence. Adversarial regressions gate deployment stages, extending the AD-34 closed loop to the release pipeline. Supplier-favoring manipulation is surfaced to compliance with a contractual review SLA. The accepted costs are detection latency, ongoing dataset maintenance (PRD-008), and a manual review queue for flagged negotiations. AD-43 (Bedrock Guardrails on all agents) is the runtime control whose effectiveness these evaluators verify; AD-34 defines the automated actions triggered by the scores they produce.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
