# AD-095 — A2A Agent Model-Tier Never-Raise Fallback

**Theme:** Dynamic Configuration & the Agent Factory
**Catalog:** AD-95 · **Source PRD:** PRD-010 · **Status:** Accepted · **Related:** AD-48, AD-49, AD-65, AD-25, AD-57

## Context

AD-65 makes `DynamicAgentFactory` the single Bedrock-request assembly point, and AD-48 makes config-plane reads fail-fast there. Originally the three standalone A2A agent runtimes (`impl/agents/*/config.py`) could not use the factory: their Docker build context could not import `agents/shared/`, so each resolved its own model **at init**, outside the factory. That created a gap AD-48 did not cover — the factory's fail-fast contract physically cannot run on this path, yet a model id must be resolved before the agent can serve A2A traffic. A transient `{env}-system-config` read failure during a cold start would, under a naive fail-fast, prevent the agent from booting at all. The model-tier labels themselves come from AD-57 (cognitive-demand tiering); what was undefined was how an off-factory agent resolves a tier label to a concrete model id when the config plane blips at init.

## Decision

On the agent init path, model-tier resolution uses a three-level precedence, a second bounded exception to the AD-48 fail-fast rule:

1. `{env}-system-config` `model` item, `tiers[<tier>]` — the live mapping (normal path);
2. `BEDROCK_MODEL_ID` environment variable — a deploy-time operator override;
3. a hardcoded default that mirrors the seed config — so a fresh environment boots.

The exception is scoped to **model-tier resolution on the agent init path**. It does not extend to thresholds, temperatures, feature flags, or to any other config value, all of which remain fail-fast (AD-48). The hardcoded default mirrors `seed_system_config.py` so the fallback can never resolve to a model the platform was not seeded with. **As built (2026-06-21, PR #42):** the standalone-vs-factory split is resolved — all 7 agents now share one `resolve_model_id` ladder inside `DynamicAgentFactory` (delivered via the immutable `agent-base` image, AD-101), so this never-raise ladder is the single model-init path. A CI drift-guard on `SEED_TIERS` and a permanent placement-guard test keep it never-raise and on the boot path.

## Alternatives Considered

- **Apply AD-48 fail-fast uniformly on the A2A init path.** Rejected: a transient config-plane read at cold start would block the agent from booting, turning a brief DynamoDB blip into an availability outage for a whole agent runtime — with no correctness benefit, since the seed-mirroring default is a known-good model id, not a guessed governance value.
- **Make all A2A agents share one factory.** **Adopted (PR #42):** rather than importing `agents/shared/`, the factory was extracted into a shared package (`impl/packages/buyer_agent_core/`) delivered via the immutable `agent-base` image (AD-101), so all 7 agents now resolve through one `DynamicAgentFactory`. This was the eventual merge previously tracked as debt; it has landed — the never-raise ladder is retained as the model-tier fallback contract, now on the single shared path.
- **Broaden the exception to all config on the A2A path (thresholds, temperatures).** Rejected: re-creates the retired `_COLD_START_DEFAULTS` problem (AD-48) — "safe" is undefined for numeric thresholds; only a model-id default is provably a known-good value.
- **Hardcode the model id directly, no system-config read.** Rejected: forfeits config-as-data (AD-25) — operators could not retune the tier→model mapping without a redeploy; the live `tiers[<tier>]` read is the primary path, the env var and hardcoded default are only the fallback ladder.

## Trade-offs

| Gained | Given up |
| --- | --- |
| An A2A agent boots through a transient config-plane blip instead of failing its cold start | A second named exception to AD-48 — readers of the fail-fast rule must now know two carve-outs (this and AD-49) |
| Operators get a deploy-time lever (`BEDROCK_MODEL_ID`) and a fresh env boots from the hardcoded seed-mirror | The hardcoded default must stay in sync with the seed; drift would let an agent boot on a stale model id |
| Config-as-data preserved — the live `tiers` map is still the primary source (AD-25, AD-57) | The ladder is now the single config read on the agent boot path, so it must stay never-raise — enforced by a CI drift-guard plus a permanent placement-guard test |

The exception is bounded to model-tier resolution because a model id has a provably-safe known default (the seed value), which numeric thresholds and governance values do not — the same boundary that limits AD-49 to feature flags.

## Results

Specified in PRD-010 §3.5 (A2A Agent Model-Tier Fallback) and REQ-C003 (updated from "one exception" to "two bounded exceptions"), shipped in PRD-003-impl v1.3.0 / PRD-010-impl v1.5.3 (impl repo PR #22). It is the second of the two bounded exceptions named by AD-48; the first is AD-49. It is a deliberate, documented exception to AD-65's "single assembly point" model-config fail-fast — the model tier (and only the model tier) falls back rather than failing the cold start. **As built (2026-06-21, PR #42):** the shared-resolver merge has landed — the factory was extracted into `impl/packages/buyer_agent_core/` and delivered via the `agent-base` image (AD-101), so all 7 agents resolve through one `DynamicAgentFactory` and this never-raise ladder is the single model-init path. There is no longer a separate off-factory code path; the decision stands as the model-tier fallback contract on the unified path.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
