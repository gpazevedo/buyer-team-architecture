# AD-055 — Five-Environment Model with Per-Environment Cedar, PITR, and Retention Settings

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-55 · **Source PRD:** PRD-007 · **Status:** Accepted · **Related:** AD-40, AD-54, AD-8

## Context

Different environments serve different purposes and carry different risk profiles. A CI sandbox should not pay for PITR or enforce Cedar denials that slow down developer iteration, while production must enforce Cedar (AD-40) and retain data durably. Without a structured per-environment policy, either lower environments are over-engineered (expensive) or production is under-protected.

## Decision

Define five environments — Local, CI, Dev, Staging, and Production — each with its own Cedar enforcement mode, PITR setting, log retention, and canary configuration. Cedar is LOG_ONLY in CI/dev/staging and ENFORCE in production only; PITR is production-only.

## Alternatives Considered

- **Two environments (staging mirrors prod exactly, dev is unrestricted).** Rejected: does not provide the cost scaling of Local/CI tiers or the intermediate validation step that Staging (manual promote, full E2E) provides before production.
- **Uniform ENFORCE Cedar across all environments.** Rejected: Cedar denials in dev and CI would block routine development work before policies are stable; LOG_ONLY preserves observability without blocking.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Progressive validation with cost and safety scaled to each stage — cheap, permissive lower environments; strict, durable production | Parity — lower environments differ from prod (LOG_ONLY Cedar, no PITR, mock/SimpleLLM Bedrock), so prod-only behaviors such as ENFORCE denials are not exercised until staging |
| LocalStack at the bottom tier lets developers iterate without AWS cost (REQ-I200) | The parity gap is mitigated by staging E2E but not eliminated; a Cedar policy error in production could escape staging validation |
| Per-environment overrides are config-driven (`policy_enforcement_mode`, `enable_pitr`, `log_retention_days`, `canary_traffic_pct`) without code changes | — |

## Results

Environment table (PRD-007 §8) fixes each property per tier. Satisfies REQ-I200 (five environments), REQ-I201 (production PITR), REQ-I202 (log retention: prod 365d / staging 90d / dev 30d), REQ-I203 (Cedar LOG_ONLY in CI/dev/staging, ENFORCE in prod), and REQ-I204 (no production data in non-production environments). The regional constraint from AD-8 applies to every non-local environment. AD-54's build-once-promote pipeline drives the promotion path through these tiers; AD-56's canary settings are applied in the production tier.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
