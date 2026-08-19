# AD-040 — Cedar Rollout Phases Distinct from Per-Environment Mode

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-40 · **Source PRD:** PRD-005 · **Status:** Accepted — scope narrowed 2026-08-06 to PO Receiving; widened 2026-08-19 to five Gateways (see AD-39's Results); phase sequencing corrected 2026-08-19 · **Related:** AD-39, AD-55, AD-147, AD-148, AD-149

> **Scope correction (2026-08-06, follow-up to AD-39's correction).** This rollout sequence was written for "Cedar enforcement" generally, at a time this ADR family assumed Cedar would eventually cover the 6 LLM agents' ~60-rule table in addition to PO Receiving. AD-39 now establishes that the 6 agents' own tool calls never traverse a Gateway, so there is no Cedar deployment to roll out there, ever — the phases below apply to PO Receiving only, the one Gateway Cedar actually fronts. The distinction this ADR draws (a one-time temporal rollout vs. a permanent per-environment mode) is still the right model; it now has a single subject rather than the "6 agents + PO Receiving" scope this ADR previously implied.

## Context

Enabling Cedar enforcement in production is risky: a too-strict policy blocks legitimate calls. But the question of how enforcement is introduced over time in production and the question of which mode each environment permanently runs in are two different concerns that are easy to conflate. Conflating them produces either a production environment that is never safely cutover, or a CI environment that behaves differently depending on where production is in its rollout sequence.

## Decision

Define a one-time temporal production rollout (Observe LOG_ONLY → Validate LOG_ONLY+alerts → Enforce) as a sequence kept explicitly separate from the permanent per-environment Cedar mode (LOG_ONLY in CI/dev/staging, ENFORCE in prod, REQ-I203). The rollout phases are a time-boxed transition; the per-environment mode is a permanent configuration.

## Alternatives Considered

- **Single unified Cedar mode per environment, promoted through environments.** Rejected: CI and staging would have to track production's rollout phase, coupling environment config to a one-time operational sequence.
- **Enable ENFORCE in production immediately.** Rejected: no observation window to detect over-permissive or under-permissive rules before they block legitimate traffic.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Safe production cutover — observe and tune in LOG_ONLY before blocking; CI stays LOG_ONLY permanently regardless of production's rollout phase | Conceptual overhead — operators must hold two independent axes (temporal rollout phase vs. permanent environment mode) in mind simultaneously |

An earlier version's section ordering in the PRD invited the confusion between these two axes; the PRD now spends a paragraph explicitly disentangling them.

## Results

A three-phase rollout table is specified in PRD-005 §5.2, scoped to PO Receiving (§9) — see AD-39's correction for why the 6 LLM agents' tool calls are out of scope for this rollout entirely, not merely earlier in it. Permanent per-environment modes are owned by PRD-007 §8 / REQ-I203 and governed by AD-55's five-environment model. The two concerns are documented as independent. Cedar policy generation and the authoritative permission table are governed by AD-39.

**Update 2026-08-19 (impl PR #333) — production's phase 1 was unreachable by construction, and the rollout now covers five Gateways, not one.**

`prod-deploy.yml` passed `receiving_policy_mode=ENFORCE` under a comment reserving the flip until "current prod LOG_ONLY telemetry has been reviewed." That precondition could never be satisfied: the plan step carrying the variable is the only thing that creates production's policy engine, so the first apply would have created it *already enforcing* — the Observe-phase telemetry its own go-ahead depended on could not exist. Both call sites (the plan step and the roll-forward apply) now pass `LOG_ONLY`, which is this ADR's phase 1 actually happening; ENFORCE becomes a separate, deliberate one-word deploy afterwards.

Two invariants fell out of the fix and belong here rather than in the workflow's comments alone:

- **The roll-forward apply must carry whatever mode the plan carries.** They had diverged. A rollback that changes enforcement posture along with the image tag breaks AD-54's premise that rollback introduces no new variables — the mode is a deploy-time `-var`, so it is exactly such a variable.
- **The phases are not ceremony; skipping to ENFORCE can leave the system less protected than LOG_ONLY.** `po_receiving.cedar` is deny-by-default on a single permit conditioned on `principal.getTag("scope")` resolving. If that tag does not resolve in production, ENFORCE denies every call — and the caller does not fail. Node 7 catches the 403 and degrades to the in-process boto3 write, so POs keep flowing while each one bypasses the Cedar engine, the REQUEST interceptor and the JWT authorizer together. This inversion is general enough to constrain the four flips still pending on the other engines, so it is recorded on its own as [AD-149](AD-149-alarm-the-bypass-before-enforce.md), along with the bypass alarm that is now the flip's prerequisite.

The rollout table itself is unchanged; what changed is that phase 1 is now the state production will actually be created in, and that the same three phases apply to `pr_ingest`, `sfn_orchestrator`, `tenant_mdm` and `master_data` (AD-39's Results, AD-148's addendum for why none of them should flip yet).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
