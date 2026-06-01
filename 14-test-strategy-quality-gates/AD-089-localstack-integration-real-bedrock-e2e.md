# AD-089 — LocalStack for Integration; Real Bedrock (SimpleLLM) for E2E

**Theme:** Test Strategy & Quality Gates  **Catalog:** AD-89 · **Source PRD:** PRD-008 · **Status:** Accepted · **Related:** AD-88, AD-90

## Context

Integration tests need realistic AWS service behavior (DynamoDB, S3, SQS) but running them against real Bedrock on every dev deployment would incur substantial inference cost and latency. At the same time, mocking the LLM for E2E tests would defeat their purpose — E2E tests exist precisely to validate real model behavior across a full negotiation. The system needs a mock strategy that is cost-efficient for the integration layer while preserving model realism at the E2E layer. Additionally, `BUYER_TEAM_TEST_MODE` must clearly separate local/mock from live modes, and the Mem0 integration path (`mem0_enabled=true`) adds a second independent toggle (`BUYER_TEAM_MEM0_MODE`).

## Decision

Use LocalStack for DynamoDB, S3, and SQS in integration tests — with real A2A between agent components inside the Docker network — and use real AWS with the cheaper SimpleLLM tier for E2E tests. Integration tests are gated at dev deploy (G5); E2E tests are gated at staging promotion (G6). `BUYER_TEAM_TEST_MODE=mock` selects LocalStack; `BUYER_TEAM_TEST_MODE=live` selects real AWS. When `mem0_enabled=true` in the staging or E2E environment, E2E tests exercise the real Mem0 endpoint.

## Alternatives Considered

- **Real AWS for integration tests.** Rejected: incurs Bedrock inference cost and AWS service latency on every dev deployment, making the integration gate too slow and expensive to be practical.
- **Mock all AWS services (no LocalStack) for integration.** Rejected: unit-level mocks (`unittest.mock`) cannot exercise cross-component behavior over A2A; real A2A within Docker is essential for contract validation.
- **Full DefaultLLM tier for E2E.** Rejected: the strongest model on every E2E run would make staging promotion prohibitively expensive; SimpleLLM provides sufficient model realism for functional validation, and DefaultLLM is used by the LLM-as-Judge evaluators at staging promotion (G7).
- **Status quo / no action.** Rejected: without an explicit mock strategy, integration tests would either use real Bedrock (too expensive) or mock everything (insufficient realism for A2A contract validation).

## Trade-offs

| Gained | Given up |
| --- | --- |
| Integration tests are realistic enough — real AWS API semantics via LocalStack and real A2A — without Bedrock inference cost | LocalStack is not byte-identical to real AWS; subtle service-behavior differences can pass locally and fail in the cloud |
| E2E still exercises a real model so model-dependent behavior is genuinely validated before staging promotion | E2E uses SimpleLLM rather than the full DefaultLLM tier on every run, so the strongest-model paths receive less continuous coverage |
| Cost is spent on model realism only where it is required (E2E and above) | The Mem0 live-endpoint path is exercised at E2E only when `mem0_enabled=true`; mock mode uses an in-memory stub, leaving real Mem0 behavior untested below staging |

The LocalStack fidelity gap is mitigated by running E2E against real AWS before staging promotion (G6) and by the full-suite nightly run. The DefaultLLM coverage gap is mitigated by the staging-promotion LLM-as-Judge evaluation suite (G7).

## Results

Integration layer is gated at dev deploy (G5) and realized via `docker-compose.test.yml`, which starts LocalStack (DynamoDB, S3, SQS) with a seeded system-config table alongside all seven agent containers and the Graph Orchestrator. E2E layer is gated at staging promotion (G6) against real AWS with SimpleLLM. `BUYER_TEAM_TEST_MODE=mock|live` toggles LocalStack vs real AWS; `BUYER_TEAM_MEM0_MODE=mock|live` is independent and controls Mem0 endpoint selection (REQ-T043). When `mem0_enabled=true`, E2E-01–04 exercise Mem0 integration points IP-1, IP-2, and IP-4. This mock strategy is a direct implementation of the gate structure defined by AD-88 and supports the golden-dataset eval path exercised by AD-90.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
