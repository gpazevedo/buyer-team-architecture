# AD-040 — Cedar Rollout Phases Distinct from Per-Environment Mode

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-40 · **Source PRD:** PRD-005 · **Status:** Accepted · **Related:** AD-39, AD-55

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

A three-phase rollout table is specified in PRD-005 §5.2. Permanent per-environment modes are owned by PRD-007 §8 / REQ-I203 and governed by AD-55's five-environment model. The two concerns are documented as independent. Cedar policy generation and the authoritative permission table are governed by AD-39.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
