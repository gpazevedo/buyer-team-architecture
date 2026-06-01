# AD-049 — Security-Critical Flag Exception to Fail-Fast

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-49 · **Source PRD:** PRD-006 · **Status:** Accepted · **Related:** AD-48, AD-25, AD-63, AD-66

## Context

AD-48 establishes that any config-plane unavailability must cause a hard fail — no fallback, no degraded agent. This rule holds for governance thresholds and model parameters, where an unknown value could silently corrupt a negotiation. Two feature flags are different in kind: `galileo_protect_enabled` and `kraljic_classification_override_enabled` are safety toggles where `false` (disabled) means enforcement is on and `true` (enabled) means a safety feature is relaxed. For these flags, failing fast on a config-plane outage leaves negotiations stalled; hard-defaulting to `false` keeps enforcement on with no loss of security posture. The broader `_COLD_START_DEFAULTS` fallback (retired v1.2.0) had been the prior mechanism; that retirement left these two flags without an explicit rule.

## Decision

`galileo_protect_enabled` and `kraljic_classification_override_enabled` are the sole exception to AD-48's no-fallback rule: when `{env}-system-config` is unreachable, both flags hard-default to `false`. A disabled flag means enforcement is on, so defaulting to `false` preserves security posture during a config-plane outage. All other configuration values continue to fail fast per AD-48.

## Alternatives Considered

- **Apply AD-48 uniformly (fail fast for these flags too).** Rejected: prevents negotiations from proceeding during a config outage even though the safe posture (enforcement on) is unambiguously known; the only reason to fail fast on config is to avoid running under an unknown or wrong value, but `false` for these flags is by definition the safe value.
- **Default to `true` (enabled/relaxed) as the fallback.** Rejected: `true` relaxes enforcement; a config-plane outage could then be exploited to silently disable a safety feature, violating the monotonic-security-posture requirement.
- **Extend the carve-out to other "safe" config values.** Rejected: "safe" is not well-defined for numeric thresholds; broadening the exception would re-create the retired `_COLD_START_DEFAULTS` problem. The exception is scoped to exactly two flags justified one-by-one.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A config-plane outage cannot be used to silently disable a safety feature; security posture is monotonic under config failure | AD-48 now has a named exception, which is a documentation and reasoning burden for every reader of the fail-fast rule |
| Negotiations can proceed during a config outage with enforcement on, rather than stalling unnecessarily | The exception must be maintained precisely — adding a third flag to this list requires the same justification as the original two |

The exception is intentionally minimal. Every reader of AD-48 must also know this carve-out; the cost is accepted because the blast radius is two flags with an explicit, auditable justification for each.

## Results

Recorded in PRD-006 §3.1 (security-critical flag exception row) and cross-referenced from PRD-005 and PRD-010 §3.3 (tombstone). The `_COLD_START_DEFAULTS` fallback was retired at v1.2.0, leaving only these two hard-defaults. In the feature-flag lifecycle (AD-66), these two flags carry the additional fail-safe default and are exempt from the standard five-phase removal contract — they remain as permanent safety controls.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
