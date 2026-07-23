# AD-127 — Build a Self-Sufficient RFC 8693 Token-Exchange Broker for the Delegation Chain

**Theme:** Security & Governance / Trust Boundaries
**Catalog:** AD-127 · **Source PRD:** PRD-005 · **Status:** Accepted (built, not enabled) · **Related:** AD-42, AD-39, AD-94

## Context

When a PR is human-approved and a PO is delivered downstream (the PO Receiving Gateway, AD-39 / PRD-013), the outbound token should carry *who acted* — both the approving human and the acting agent — so a delegated action is attributable end-to-end rather than appearing to originate from an anonymous machine identity. This is the RFC 8693 (OAuth 2.0 Token Exchange) delegation chain, and it was PRD-005's first open identity gap (surfaced by a comparison against AWS's "Loom for AWS" reference agent platform).

Two facts, confirmed by direct inspection, ruled out the obvious approaches:

- **Cognito's OAuth2 endpoint supports only four grant types** — authorization-code, implicit, client-credentials, refresh-token. It does **not** implement `urn:ietf:params:oauth:grant-type:token-exchange`, so it cannot mint a delegated token at all.
- **AgentCore Identity's on-behalf-of (OBO) exchange only *forwards*** to a downstream IdP's own token endpoint. Cognito exposes no such endpoint for this flow, so adopting AgentCore Identity would provision resources without actually performing the exchange.

Meanwhile the human approver's identity was *already* being captured — `node_approval_gate.py` persists `approved_by` — but nothing downstream read it, so the delegated action was un-attributable even though the data existed.

## Decision

Build a small, self-sufficient authorization server: `lambdas/token_exchange_broker`, with its own OIDC discovery document and JWKS. It verifies an inbound Cognito-signed M2M token against Cognito's own JWKS, then mints its **own** RS256-signed token that adds **additive** `actingAgent` / `actingUser` claims. The Cedar principal (`sub`) is deliberately **left unchanged**, so `po_receiving.cedar` and its existing fixtures require no modification. The broker is wired through `po_delivery.py`, `pr_event_router`, and `gateway_interceptor`, and the persisted `approved_by` is now threaded downstream (`node_award_comms._deliver_to_receiving` → `po_delivery.deliver_purchase_order` as `acting_user_sub`; the `"system:auto_approval"` sentinel normalises to `None`).

Every consumer is gated behind an **off-by-default** flag (`TOKEN_EXCHANGE_MODE`, `REQUIRE_ACTING_AGENT`). The broker ships as code plus `infra/token_exchange_broker.tf` (validated, **never applied**); no trust boundary has been cut over. This is an intentional "build the mechanism, defer the cutover" posture consistent with AD-42's dormant-then-wire pattern.

## Alternatives Considered

- **Use Cognito's token endpoint for the exchange.** Rejected: Cognito supports only 4 grant types, none of them `token-exchange`.
- **Adopt AgentCore Identity OBO.** Rejected: it only forwards to a downstream IdP's token endpoint, which Cognito doesn't expose here — it would add resources without performing the exchange.
- **Overwrite the Cedar principal (`sub`) with the acting identity.** Rejected: it would change the authorization subject, forcing a rewrite of `po_receiving.cedar` and all its fixtures; additive claims keep the existing policy and tests intact.
- **Status quo (no delegation).** Rejected: the approving human's identity (`approved_by`) was captured but dropped before the downstream call, leaving delegated PO actions un-attributable.

## Trade-offs

| Gained | Given up |
| --- | --- |
| End-to-end attribution of a delegated action (acting human + acting agent) without changing the authorization subject | A second signing authority to operate (own key, discovery, JWKS) rather than reusing Cognito |
| Existing Cedar policy and fixtures untouched — additive claims, unchanged `sub` | The Cedar `actingAgent` clause that would *consume* the new claims can't be enabled yet (see Results) |
| Mechanism lands independently of the risky trust-boundary cutover (off-by-default flags, TF unapplied) | The delegation chain provides no live guarantee until the flags flip and the broker is deployed — it is built, not enabled |

## Results

Shipped in impl PR #223 (merged 2026-07-18) as code + Terraform, all behind off-by-default flags; documented as PRD-005 §4.2 (as-built note). A pre-existing cold-start bug found and fixed in the same PR: two concurrent broker cold starts racing `create_secret` would raise an uncaught `ResourceExistsException` — the broker now reads back the winner's key rather than each container minting its own untrusted signing key. The `lambdas/` handler test suites (including this broker) were also wired into `pr-checks.yml` for the first time.

**Deliberately deferred — the actual trust-boundary cutover, none done:**

- Repointing a Gateway's `discoveryUrl` at the broker.
- Enabling the Cedar `actingAgent` policy clause. A proposed-but-unapplied test (`infra/policies/test_po_receiving_acting_agent_clause_proposed.py`) proves that adding the clause *today* would flip `test_allow_with_receive_scope` — and real traffic of that shape — from Allow to Deny, because no caller yet carries the `actingAgent` tag. It needs the live JWT-claim → Cedar-principal-tag mapping confirmed in dev first.
- Flipping `TOKEN_EXCHANGE_MODE` / `REQUIRE_ACTING_AGENT`, and `terraform apply` of `infra/token_exchange_broker.tf`.

The CI build gap this introduced — `data.archive_file.token_exchange_broker` fails `terraform plan` unless the zip is built first — was fixed in PR #224 (a build step added to every job that runs an untargeted plan/apply). This ADR closes one of the three "Loom for AWS" gaps; the other two (mandatory resource tagging, lightweight agent registry) are convention/tooling additions documented in PRD-007, not architectural decisions.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
