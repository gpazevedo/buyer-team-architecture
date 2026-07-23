# AD-128 — Precomputed Context Renders After the Cache Point, and Anonymize-Only Guardrail Interventions Are Not Blocks

**Theme:** Security & Governance / Trust Boundaries
**Catalog:** AD-128 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-43, AD-35, AD-28, AD-65

## Context

`strategic_partnership` and `bottleneck_negotiation` reproducibly failed real invokes with **"Your input contains blocked content."** in ~4s — a Bedrock Guardrail rejection before any work started. The same `GUARDRAIL_ID` is wired to every agent (AD-43), so this was not a per-agent misconfiguration; something in these two agents' request *content* tripped the filter. This is precisely the "gap concentrated in specific agents or the request-shape used to reach them" that AD-43's Results left open after AD-35's adversarial re-run scored only 30%.

The root cause is **two coupled bugs** — fixing only the first just moves the failure to the second:

1. **Input false-positive.** These two agents' large orchestrator-precomputed blobs (`relationship_histories` / `risk_assessments` / `tco_results`) sit in the **user turn**, where they score MEDIUM on `PROMPT_ATTACK`. At the `PROMPT_ATTACK:HIGH` strength AD-43 mandates, the Guardrail blocks even a LOW-confidence hit — so legitimate business context read as an attack.
2. **Output false-positive.** `serve.py` raised on **any** `stop_reason == "guardrail_intervened"`. But `EMAIL`/`NAME`/`PHONE`/`ADDRESS` are configured `ANONYMIZE`, not `BLOCK`, in `infra/modules/security/main.tf` — deliberately, so routine negotiation content (supplier emails the model echoes) doesn't break the flow. Output PII masking therefore raised `guardrail_intervened` and was treated as a hard failure.

The obvious fix for bug 1 — lowering `PROMPT_ATTACK` strength — was rejected outright: it regresses the AD-35 adversarial catch rate fleet-wide from 100% to 55%.

## Decision

**Bug 1 — render precomputed blobs into the system role, after the cache point.** New `AgentSpec.context_builder` + `factory._system_prompt_for` place the precomputed blobs into the **system role, after the `cachePoint`**. The static cache prefix stays byte-identical (the AD-65 / AD-28 REQ-C004 invariant holds — only the variable data lands after the breakpoint, uncached), and the system role is **not** `PROMPT_ATTACK`-scanned, so the blobs no longer trip the filter.

This is **scoped to the precomputed blobs only.** `items` / `suppliers` deliberately stay in the user turn: the AD-35 adversarial suite (`evals/adversarial_robustness.py::_build_request`) injects its attack payload into `items`, rendered by `_build_prompt`. Moving `items` into the (unscanned) system role would silently drop these two agents' prompt-injection coverage — the exact regression the strength decision above was protecting.

**Bug 2 — distinguish an anonymize-only mask from a real block.** New `guardrail.is_masking_only_intervention(trace)` classifies the intervention: `serve.py` re-raises only on a real block and lets an anonymize-only masking pass through. It is **secure by default** — anything not affirmatively anonymize-only is treated as a block — mirroring Strands' own `_find_detected_and_blocked_policy`. The Guardrail trace isn't on `AgentResult`; it rides a raw `metadata.trace.guardrail` stream chunk, captured inline in the executor stream and via a `callback_handler` on the `run()` path (`guardrail_trace_of`).

## Alternatives Considered

- **Lower `PROMPT_ATTACK` strength from HIGH.** Rejected: regresses the AD-35 adversarial catch rate 100% → 55% fleet-wide.
- **Move `items`/`suppliers` into the system role too (uniform treatment).** Rejected: the system role isn't `PROMPT_ATTACK`-scanned and the AD-35 suite injects into `items`, so this would silently blind these agents to prompt injection.
- **Treat every `guardrail_intervened` as success (invert the gate).** Rejected: a real BLOCK would then pass through — the gate must stay secure-by-default, blocking on anything not provably anonymize-only.
- **Reconfigure PII entities from ANONYMIZE to BLOCK for consistency.** Rejected: BLOCK on routine supplier emails/names is exactly the flow-breaking behavior the ANONYMIZE choice in `module.security` was made to avoid.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Two agents unblocked without weakening `PROMPT_ATTACK:HIGH` or the AD-35 catch rate | Per-agent split of what lands in system vs. user role — a `context_builder` seam agents must opt into correctly, or context silently stays in the scanned user turn |
| Cache-prefix invariant (AD-65/AD-28) preserved — variable context sits after the `cachePoint`, static prefix unchanged | Guardrail-trace capture depends on a raw `metadata.trace.guardrail` stream chunk, not a stable `AgentResult` field — coupled to Strands' streaming shape |
| Output masking no longer fails the run; secure-by-default classification blocks anything not provably anonymize-only | Adds a classification path that must be kept in step with the guardrail's ANONYMIZE/BLOCK policy config if that config changes |

## Results

Shipped in impl PR #226 (merged 2026-07-19). Files: `buyer_agent_core/guardrail.py` (`is_masking_only_intervention` + `guardrail_trace_of`), `serve.py` (masking-aware gate on both the `run()` and streaming-executor paths), `spec.py` / `factory.py` (`context_builder` + per-request system context after the `cachePoint`), and `agents/{strategic_partnership,bottleneck_negotiation}_llm/agent_spec.py` (split precomputed blobs → system, params → user turn). 411 core + 36 eval tests pass, with new block/anonymize coverage at unit, `run()`, and streaming-executor levels, and factory context-after-`cachePoint` tests; the AD-35 contract (inject into `items`, rendered by `_build_prompt`) is preserved.

**Not yet live-verified (requires Bedrock).** Three claims remain to prove on dev: (1) the block clears with *only* the precomputed blobs moved (the fully-neutral-message case was proven live; keeping `items`/`suppliers` in the user turn is a slightly different shape); (2) AD-35 still scores 100% for these two agents; (3) the tool loop completes when output PII is masked mid-loop. This partially answers AD-43's open "concentrated in specific agents or request-shape" question — the block was request-shape (precomputed blobs in the scanned user turn), not an agent-identity misconfiguration.

This decision is downstream of AD-43 (which establishes the Guardrail and its `PROMPT_ATTACK:HIGH` + ANONYMIZE-PII policy) and constrained by AD-65/AD-28 (the cache-prefix invariant it must not break).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
