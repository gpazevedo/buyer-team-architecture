# AD-088 — Five-Layer Test Pyramid with Distinct Gates per Layer

**Theme:** Test Strategy & Quality Gates  **Catalog:** AD-88 · **Source PRD:** PRD-008 · **Status:** Accepted · **Related:** AD-89, AD-90

## Context

Tests vary enormously in cost and speed — a unit test runs in milliseconds against mocks, an E2E negotiation runs minutes against real Bedrock, a load test runs tens of minutes. Running everything at every stage would make the feedback loop unusably slow; running too little at the PR stage allows regressions to reach expensive deployment stages. The system spans seven agents, a deterministic graph orchestrator, eight steering hooks, and Cedar policies — each with a different failure mode. Without a taxonomy that assigns each failure mode to the cheapest stage that can detect it, either developers wait too long for feedback or defects reach staging.

## Decision

Adopt a five-layer pyramid (Unit → Integration → E2E → Load → Chaos) where each layer attaches to a distinct gate: Unit blocks PR merge (G1), Integration blocks dev deploy (G5), E2E blocks staging promotion (G6), Load is advisory and runs nightly, and Chaos is deferred to v1.1. The pyramid shape is 38 unit / 56 integration / 7 E2E / 5 load / 6 chaos (future), totalling 105 test cases in v1.0 (111 with v1.1 chaos). Each PRD remains the authoritative source for its own test definitions; PRD-008 owns the taxonomy, ordering, and gating rules.

## Alternatives Considered

- **Single gate at staging.** Rejected: all tests would run on every staging promotion, making PR feedback unavailable and staging pipelines excessively slow.
- **Flat suite — same tests at every stage.** Rejected: eliminates the fast-feedback benefit; unit-test speed is lost if integration and E2E tests run on every commit.
- **Status quo / no action.** Rejected: without explicit layer-to-gate assignments, different teams would gate at different stages, producing inconsistent coverage and unpredictable pipeline durations.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Fast PR feedback — unit + lint + evals complete in < 5 min so the author has immediate signal while the change is fresh | Late detection of integration-only bugs — they surface at dev deploy (G5), not at the PR |
| Each gate fails fast at the cheapest stage that can catch a given defect class | Late detection of end-to-end bugs — they surface at staging promotion (G6) |
| Cost is amortized — expensive E2E and load tests run only when a change is already validated at cheaper layers | No automated resilience-under-failure coverage in v1.0; chaos / fault-injection is explicitly deferred to v1.1 |

The pyramid shape deliberately accepts that the broad, cheap base (unit + integration) catches most defects and the narrow, expensive top (E2E + load) catches the remainder. The v1.1 chaos layer (AWS FIS injections CH-01–06) will close the fault-injection gap.

## Results

Eight CI/CD gates (G1–G8) map to the pyramid layers and are realized in the GitHub Actions pipeline defined in PRD-008-test-strategy-impl.md. G1 (unit), G3 (Strands Evals, see AD-90), G4 (security scans), and G4a (image size) block PR merge; G5 (integration) blocks dev deploy; G6 (E2E) and G7 (full eval suite) block staging promotion; G8 (prod smoke) triggers rollback. The test-environment split — LocalStack for integration, real Bedrock for E2E — is realized by AD-89. Agent-quality regression detection at the PR is realized by AD-90. The 105-case inventory is owned by PRD-008 §9 with contributions from PRD-002, PRD-003, PRD-004, PRD-005, PRD-006, PRD-010, and PRD-014.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
