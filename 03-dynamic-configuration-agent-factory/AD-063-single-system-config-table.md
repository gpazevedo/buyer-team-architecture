# AD-063 — Single `{env}-system-config` Table, Read-Once-at-Instantiation

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-63 · **Source PRD:** PRD-010 · **Status:** Accepted · **Related:** AD-25, AD-48, AD-49, AD-64, AD-65, AD-66

## Context

Model IDs, governance thresholds, feature flags, and external-service rates must be changeable without redeploying code. An earlier design used AWS AppConfig with gradual rollout, sidecar caching, and auto-rollback Lambda — adding an extra sidecar process, a polling loop, and operational surface that ultimately did not prevent the "running on stale config" failure mode. AppConfig was retired at platform v1.7.0 (PRD-005 §5.3, PRD-010 §6) and its integrity controls (IAM `StartDeployment` conditions, EventBridge rules, auto-rollback Lambda) were replaced by DynamoDB write-deny resource policies and `system_config.change` alarms.

## Decision

All dynamic configuration lives in a single `{env}-system-config` DynamoDB table with four config groups: `governance`, `model`, `features`, and `external-rates`. The table is read once per agent instantiation — no polling, no sidecar, no mid-session change. A value changed in DynamoDB takes effect on the next instantiation.

## Alternatives Considered

- **AWS AppConfig with sidecar caching and gradual rollout.** Rejected: adds a sidecar, polling, and operational surface; its integrity controls were less direct than DynamoDB resource policies; retired at platform v1.7.0.
- **Environment variables baked into the container image.** Rejected: requires a redeploy for any parameter change; defeats the purpose of runtime-configurable config.
- **Per-service config tables (one per agent or subsystem).** Rejected: fragments the config surface, increases IAM complexity, and makes cross-agent consistency harder to audit; a single table with group-keyed items is sufficient.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Simplicity — one table, no sidecar, no polling loop; config is plain data, queryable and IAM-protectable | AppConfig's gradual-rollout and auto-rollback machinery; config changes are all-or-nothing on next read with no built-in canary for config itself |
| A value changed in DynamoDB takes effect on the next instantiation, with no redeploy | Read-once means an in-session agent never sees a mid-flight config change; urgent changes wait for the next instantiation (intentional — stable behaviour per negotiation) |
| Config changes are auditable via CloudTrail data events and `system_config.change` alarms | The `external-rates` group is read only by the monthly cost report, not the agent path — its freshness matters differently |

The absence of a built-in gradual-rollout mechanism for config changes is mitigated by the feature-flag lifecycle (AD-66) for behavioral changes and by the override-table hardening (PRD-010 §5.1 / AD-44) for threshold changes.

## Results

`DynamicAgentFactory` reads this table at instantiation (AD-65, PRD-010 §3). The fail-fast rule (AD-48) applies if the table is unreachable, with the two security-critical flag exceptions (AD-49). Two-stage threshold resolution (AD-64) cascades from per-tenant overrides through the profile SK to system defaults. Feature flags are evaluated at instantiation per the five-phase lifecycle (AD-66). The `external-rates` group is consumed only by the monthly per-tenant cost report (PRD-009 REQ-CST012), not by the agent runtime path.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
