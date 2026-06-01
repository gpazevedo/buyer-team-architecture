# AD-019 — Entity Access Control Only at the Interrupt-Resume API

**Theme:** Security, Governance & Trust Boundaries  **Catalog:** AD-19 · **Source PRD:** PRD-002 · **Status:** Accepted · **Related:** AD-6, AD-37, AD-18, AD-39

## Context

User-action authorization (who may approve an award, resolve a REQUIRES_ATTENTION case, or read a PR) is a fundamentally different plane from agent→tool authorization (which workload may call which tool ARN, enforced by Cedar at the Gateway). Agents act on behalf of the system, not the end user, and must never carry user JWTs. Without a clear enforcement point, user-authz checks could leak into agents or be omitted entirely.

## Decision

Enforce user-action entity access control exclusively at the Graph's interrupt-resume API. When a human approves/rejects at Node 6 or resolves a REQUIRES_ATTENTION case, the Graph validates the caller's JWT, resolves the effective `ClaimRequirement` list (per-PR override, else tenant default), AND-aggregates the checks, and accepts or returns HTTP 403 (REQ-G250/251/252). Agents do not carry user JWTs, do not evaluate user claims, and do not embed claim specs in A2A payloads. This is a separate plane from the agent→tool Cedar policies of PRD-005 §5.1.

## Alternatives Considered

- **Distribute user-authz checks across agents.** Rejected: would require pushing user JWTs into agents, expanding the attack surface and coupling agents to user identity.
- **Single global gate with no per-PR override.** Rejected: real deployments need per-PR claim overrides bounded by a tenant default.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Single testable enforcement point — user JWTs never enter the agent runtime, keeping the agent attack surface narrow | The single gate must never be bypassable; its failure is a complete authorization gap |
| Clean plane separation — agents stay free of user-identity concerns | The claim-resolution chain (per-PR override → tenant default → 403 if neither configured) is added logic to maintain |

Concentrating all user-action authorization at one API boundary is justified because it prevents user JWTs from ever entering the agent runtime, keeping prompt-cache prefixes clean and agents stateless with respect to user identity.

## Results

`POST /negotiations/{id}/approval` is the sole enforcement point for `po:approve`; agent-generated drafts carry a claim reference for routing but enforce nothing. Agents stay entirely out of user-authorization, which keeps prompt-cache prefixes clean (AD-28) and the agent attack surface narrow. Authorization failures return HTTP 403 with a clear reason; unconfigured claims fail closed (`claims_not_configured`). This plane is distinct from and complementary to agent→tool Cedar enforcement (AD-39) and node-level governance code (AD-18).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
