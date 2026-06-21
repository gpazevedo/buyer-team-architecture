# AD-095 — A2A Agent Model-Tier Fallback on the Standalone Init Path

**Theme:** Dynamic Configuration & the Agent Factory
**Catalog:** AD-95 · **Source PRD:** PRD-010 · **Status:** Accepted · **Related:** AD-48, AD-49, AD-65, AD-25, AD-57

## Context

AD-65 makes `DynamicAgentFactory` the single Bedrock-request assembly point, and AD-48 makes config-plane reads fail-fast there. But the three standalone A2A agent runtimes (`impl/agents/*/config.py`) cannot use the factory: their Docker build context cannot import `agents/shared/`, so each resolves its own model **at init**, outside the factory. That created a gap AD-48 did not cover — the factory's fail-fast contract physically cannot run on this path, yet a model id must be resolved before the agent can serve A2A traffic. A transient `{env}-system-config` read failure during a cold start would, under a naive fail-fast, prevent the agent from booting at all. The model-tier labels themselves come from AD-57 (cognitive-demand tiering); what was undefined was how an off-factory agent resolves a tier label to a concrete model id when the config plane blips at init.

## Decision

On the standalone A2A agent init path **only**, model-tier resolution uses a three-level precedence, a second bounded exception to the AD-48 fail-fast rule:

1. `{env}-system-config` `model` item, `tiers[<tier>]` — the live mapping (normal path);
2. `BEDROCK_MODEL_ID` environment variable — a deploy-time operator override;
3. a hardcoded default that mirrors the seed config — so a fresh environment boots.

The exception is scoped to **model-tier resolution on the A2A agent init path**. It does not extend to thresholds, temperatures, feature flags, or to model config on the canonical `DynamicAgentFactory` path, all of which remain fail-fast (AD-48). The hardcoded default mirrors `seed_system_config.py` so the fallback can never resolve to a model the platform was not seeded with. The two model-init paths (factory and standalone) read the same `tiers` map and await a merge into one shared resolver — tracked debt, not a behavioral divergence.

## Alternatives Considered

- **Apply AD-48 fail-fast uniformly on the A2A init path.** Rejected: a transient config-plane read at cold start would block the agent from booting, turning a brief DynamoDB blip into an availability outage for a whole agent runtime — with no correctness benefit, since the seed-mirroring default is a known-good model id, not a guessed governance value.
- **Make the A2A agents import `agents/shared/` and use `DynamicAgentFactory`.** Rejected (for now): the Docker build context does not include `agents/shared/`; restructuring the build to share the factory is the eventual merge, but it is larger than the init-path fallback and is tracked as debt.
- **Broaden the exception to all config on the A2A path (thresholds, temperatures).** Rejected: re-creates the retired `_COLD_START_DEFAULTS` problem (AD-48) — "safe" is undefined for numeric thresholds; only a model-id default is provably a known-good value.
- **Hardcode the model id directly, no system-config read.** Rejected: forfeits config-as-data (AD-25) — operators could not retune the tier→model mapping without a redeploy; the live `tiers[<tier>]` read is the primary path, the env var and hardcoded default are only the fallback ladder.

## Trade-offs

| Gained | Given up |
| --- | --- |
| An A2A agent boots through a transient config-plane blip instead of failing its cold start | A second named exception to AD-48 — readers of the fail-fast rule must now know two carve-outs (this and AD-49) |
| Operators get a deploy-time lever (`BEDROCK_MODEL_ID`) and a fresh env boots from the hardcoded seed-mirror | The hardcoded default must stay in sync with the seed; drift would let an agent boot on a stale model id |
| Config-as-data preserved — the live `tiers` map is still the primary source (AD-25, AD-57) | Two model-init code paths exist until the shared-resolver merge lands (tracked debt) |

The exception is bounded to model-tier resolution because a model id has a provably-safe known default (the seed value), which numeric thresholds and governance values do not — the same boundary that limits AD-49 to feature flags.

## Results

Specified in PRD-010 §3.5 (A2A Agent Model-Tier Fallback) and REQ-C003 (updated from "one exception" to "two bounded exceptions"), shipped in PRD-003-impl v1.3.0 / PRD-010-impl v1.5.3 (impl repo PR #22). It is the second of the two bounded exceptions named by AD-48; the first is AD-49. It is a deliberate, documented exception to AD-65's "single assembly point" — the standalone A2A runtimes are the one place an agent's model is resolved off the factory — and it reverses, for the A2A path only, the v1.4.1 narrowing that had briefly made factory fail-fast the single model-config contract. The shared-resolver merge that would fold the two init paths back together is tracked debt.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
