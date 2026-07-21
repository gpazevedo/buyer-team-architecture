# AD-049 — Feature-Flag Safe-Defaults Exception to Fail-Fast (Security-Critical Flags Primary)

**Theme:** Dynamic Configuration & the Agent Factory  **Catalog:** AD-49 · **Source PRD:** PRD-006 · **Status:** Accepted (scope broadened at PRD-010 v1.4.9 — all feature flags, not only the security-critical one) · **Related:** AD-48, AD-25, AD-63, AD-66, AD-95

## Context

AD-48 establishes that any config-plane unavailability must cause a hard fail — no fallback, no degraded agent. This rule holds for governance thresholds and model parameters, where an unknown value could silently corrupt a negotiation. Feature flags are different in kind: one in particular — `kraljic_classification_override_enabled` — is a safety toggle where `false` (disabled) means enforcement is on and `true` (enabled) means a safety feature is relaxed. For this, failing fast on a config-plane outage leaves negotiations stalled; hard-defaulting to `false` keeps enforcement on with no loss of security posture. The broader `_COLD_START_DEFAULTS` fallback (retired v1.2.0) had been the prior mechanism; that retirement first left the security-critical flag without an explicit rule, then (PRD-010 v1.4.8 → v1.4.9) the rule was widened: rather than special-casing that one flag while every other flag access used a fragile implicit `.get()` default, the factory falls back **all** feature flags to explicit safe defaults.

## Decision

When `{env}-system-config` is unreachable at instantiation, **all feature flags in the `features` config group fall back to explicit safe defaults that match the seeded defaults in `seed_system_config.py FEATURE_FLAGS`** — not only the security-critical flag. The security-critical flag (`kraljic_classification_override_enabled`) remains the **primary motivation and the load-bearing case**: its `false` default is the enforcement-maximum position, so a config-plane outage cannot relax a safety control. The broadening to all flags is a robustness choice (one explicit safe-defaults dict rather than scattered implicit defaults), and the safe default for every flag is its seed value. This is the first of two bounded exceptions to AD-48's no-fallback rule; the second is the A2A agent model-tier fallback (AD-95). All non-flag configuration — model IDs (on the canonical factory path), governance thresholds, temperatures — continues to fail fast per AD-48.

## Alternatives Considered

- **Apply AD-48 uniformly (fail fast for feature flags too).** Rejected: prevents negotiations from proceeding during a config outage even though the safe posture (enforcement on for the security-critical flags; seed value for the rest) is unambiguously known.
- **Default the security-critical flags to `true` (enabled/relaxed).** Rejected: `true` relaxes enforcement; a config-plane outage could then be exploited to silently disable a safety feature, violating the monotonic-security-posture requirement.
- **Special-case only the two security-critical flags; leave other flags on implicit `.get()` defaults.** Rejected (the prior v1.4.8 framing): scattering implicit defaults across each flag access site is fragile and easy to get wrong; one explicit safe-defaults dict matching the seed is auditable and uniform. The security-critical pair still carries the explicit justification.
- **Extend the carve-out to numeric thresholds / model IDs.** Rejected: "safe" is not well-defined for thresholds; broadening past feature flags would re-create the retired `_COLD_START_DEFAULTS` problem.

## Trade-offs

| Gained | Given up |
| --- | --- |
| A config-plane outage cannot be used to silently disable a safety feature; security posture is monotonic under config failure | AD-48 now has named exceptions, a documentation and reasoning burden for every reader of the fail-fast rule |
| Uniform, auditable flag fallback — one explicit safe-defaults dict matching the seed, rather than fragile per-site implicit defaults | The safe-defaults dict must stay in sync with `seed_system_config.py FEATURE_FLAGS`; drift would let a flag fall back to a stale value |
| Negotiations proceed during a config outage with enforcement on, rather than stalling unnecessarily | The boundary (flags yes, thresholds/models no) must be maintained precisely so the exception never creeps into value config |

The exception is bounded to the `features` group. Every reader of AD-48 must also know this carve-out; the cost is accepted because the safe default for each flag is its seed value, with the two security-critical flags carrying an explicit, auditable justification.

## Results

Recorded in PRD-010 §3.3 ("Feature-Flag Fallback (Safe Defaults)") and REQ-C003, with the security-critical flag row in PRD-006 §3.1 and the cross-reference from PRD-005 §11.1. The `_COLD_START_DEFAULTS` fallback was retired at v1.2.0; the safe-defaults dict (PRD-010 v1.4.9) is its bounded, flags-only successor. In the feature-flag lifecycle (AD-66), the two security-critical flags carry the additional fail-safe default and are exempt from the standard five-phase removal contract — they remain permanent safety controls. This is one of the two bounded exceptions referenced by AD-48; the other is AD-95.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
