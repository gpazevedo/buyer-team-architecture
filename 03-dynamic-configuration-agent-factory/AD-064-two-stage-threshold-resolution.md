# AD-064 — Two-Stage Threshold Resolution

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-64 · **Source PRD:** PRD-010 · **Status:** Accepted · **Related:** AD-63, AD-44, AD-65, AD-25

## Context

Tenants have different risk tolerances: a risk-averse financial-services tenant may need a lower `auto_award_below_threshold_usd` and stricter supplier filters than the platform default. The platform still needs sane system-wide defaults and reusable named risk profiles so most tenants do not need per-key overrides. A single flat config makes per-tenant tuning impossible without redeploy; a fully per-tenant config table with no shared defaults creates an unbounded override surface that is hard to harden. The `{env}-tenant-evaluation-config` table is also a privilege-escalation risk (ATLAS: Persist — Modify Agent Config) because a compromised identity that can write it can lower thresholds to near-zero (PRD-010 §5.1).

## Decision

Resolve evaluation thresholds in two stages, highest priority first: per-tenant DynamoDB overrides in `{env}-tenant-evaluation-config` → system-config profile (`conservative` / `default` / `aggressive`, selected by the tenant's `config_profile` attribute via the SK in `{env}-system-config`) → system config defaults. Undefined thresholds cascade down. Override precedence operates independently at each key.

## Alternatives Considered

- **Single global config with no per-tenant overrides.** Rejected: cannot accommodate tenants with legitimately different risk tolerances without a redeploy or a one-size-fits-all policy that is too permissive for some and too restrictive for others.
- **Fully per-tenant config (no shared profiles).** Rejected: every tenant must be individually configured; no shared baseline; increases the override table size and the attack surface without a corresponding benefit.
- **Three-stage resolution (override → profile → env-level defaults → code defaults).** Rejected: the prior version included a code-defaults layer that was removed because it reintroduced hardcoded fallback values inconsistent with the fail-fast contract (AD-48); two stages — override and profile — are sufficient.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Per-tenant tuning without redeploy; most tenants need only a profile selection, not per-key overrides | Two resolution stages plus cascade: the effective threshold for a tenant is computed, not directly readable — harder to reason about at a glance |
| Named profiles (`conservative` / `default` / `aggressive`) let the platform ship sensible risk presets | The per-tenant override table is a privilege-escalation surface that requires hardening (AD-44: write-deny resource policy, CloudTrail data events, anomaly alarm, Streams revert Lambda) |
| Override precedence is per-key and independent, giving fine-grained control | Order matters at each key; a reviewer must know the cascade to understand the effective config |

## Results

Resolved thresholds attach to agent metadata at instantiation and are consumed by evaluators and steering hooks for the session duration — they are never threaded into the `system` prompt (which would fragment the cache per tenant; see AD-65). Updates take effect on the next instantiation per the read-once contract (AD-63). The override table is hardened per PRD-010 §5.1 and AD-44. The resolution logic lives in `DynamicAgentFactory._resolve_evaluation_thresholds` (PRD-010 §3.1).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
