# AD-007 — Standardize on Python/Strands/AgentCore/Terraform Stack with DynamoDB Dynamic Configuration

**Theme:** Infrastructure, Deployment & Platform Stack  **Catalog:** AD-7 · **Source PRD:** PRD-001 · **Status:** Accepted · **Related:** AD-51, AD-52, AD-63, AD-48, AD-49

## Context

The platform needs a coherent technology stack that provides a managed agent runtime and allows behavior (model selection, feature flags, thresholds) to change without redeploying. A latest-API mandate applies across the stack. The initial approach used AWS AppConfig for dynamic configuration, but it required a sidecar and added operational complexity.

## Decision

Standardize on Python 3.14, the Strands Agents SDK, Amazon Bedrock AgentCore (six services: Runtime, Memory, Gateway, Identity, Observability, Evaluations), Terraform, and GitHub Actions. Dynamic configuration is read from a single DynamoDB `{env}-system-config` table at agent instantiation via DynamicAgentFactory.

## Alternatives Considered

- **AWS AppConfig for dynamic configuration.** Rejected: required a polling sidecar, added operational complexity, and was retired at platform v1.7.0 in favor of the simpler DynamoDB table.
- **Status quo / no standardized stack.** Rejected: without a managed runtime and unified configuration plane, agent deployment and behavioral governance would require significant bespoke infrastructure.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Managed runtime removes large operational burden for session lifecycle, scaling, and microVM isolation | Maturity headroom — Python 3.14 and early AgentCore carry availability and stability risk |
| Access to newest platform capabilities (Evaluations, Cedar, AgentCore Identity) without custom plumbing | Portability — the stack is firmly coupled to AWS/Bedrock |
| Configuration treated as data, changeable without redeploy | Terraform provider coverage for AgentCore is incomplete as of 2026-05 (addressed by AD-52) |

The "bleeding-edge" risk is explicit in assumption ASM-001: the platform requires AgentCore GA in at least one US region with all six services.

## Results

AgentCore is provisioned across two Terraform providers plus SDK provisioners for the gaps (Evaluations, Cedar, Datasets), documented as explicit REQ-I001 exceptions with drift smoke tests (AD-52). Config reads are fail-fast with no stale cache and no hardcoded fallback — an agent must never run on guessed configuration (AD-48) — with two security-critical flags that hard-default to `false` to preserve security posture during a config-plane outage (AD-49). The DynamicAgentFactory is the single owner of the cache-prefix invariant and request assembly (AD-65). The AppConfig approach survives only as a tombstone in the retired-decisions catalog (AD-63 details the config table structure).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
