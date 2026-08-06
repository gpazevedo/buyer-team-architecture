# AD-040 — Cedar Rollout Phases Distinct from Per-Environment Mode

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-40 · **Source PRD:** PRD-005 · **Status:** Accepted — scope narrowed 2026-08-06 to PO Receiving; see AD-39's correction · **Related:** AD-39, AD-55

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

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
