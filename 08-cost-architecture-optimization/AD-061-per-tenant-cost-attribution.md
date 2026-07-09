# AD-061 — Per-Tenant Cost Attribution via CUR-Joined Table

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-61 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-82, AD-13, AD-57, AD-58

> **Correction (2026-07-08):** the pipeline this ADR describes — the hourly EventBridge Lambda, the CUR/Athena Glue-table join, the CloudWatch Logs Insights per-tenant token query — was never built; there is no such Lambda or Athena query anywhere in `buyer-team-impl`. The `{env}-tenant-cost-attribution` and `{env}-tenant-cost-budget` DynamoDB tables *do* exist (`infra/modules/dynamodb/main.tf:259–274`), but with the platform's generic `pk`/`sk` schema, not the composite `tenant_id` / `{date_hour}#{model_id}` key this ADR specifies — and no code anywhere writes or reads either table. They are provisioned, orphaned infrastructure. The only real per-tenant-adjacent cost signal today is negotiation-level (not tenant-level) `negotiation.total_cost_usd`, computed from `resilience/pricing.py`'s baked-in rate table and accumulated per negotiation (PRD-009 v2.0.0 §6.1) — there is no per-tenant rollup, no CUR join, and no monthly report. The body below is preserved as the original (unbuilt) decision record.

## Context

A multi-tenant platform needs accurate per-tenant spend for chargeback or per-tenant pricing decisions. Per-token estimated cost (computed from pricing constants baked into emission code at invocation time) is good enough for real-time signals but is not billing-accurate — it misses cross-region inference detours (AD-7, PRD-009 §7), tier discounts, free-tier offsets, and commercial commitment discounts. The AWS Cost and Usage Report (CUR) is the authoritative invoice source, but CUR carries no `tenant_id` dimension — it reflects total account spend per model and usage type, not per-tenant spend. Per-tenant token counts are available from CloudWatch metrics (emitted by the shared invocation wrapper, AD-13), but those use estimated pricing constants rather than CUR rates. Without a bridge between the two sources, the monthly per-tenant cost report (REQ-CST011) cannot be billing-accurate.

## Decision

Run an hourly EventBridge-scheduled Lambda that: (1) reads total Bedrock spend over the window from the CUR Glue table via Athena, normalized to tier label (SimpleLLM / DefaultLLM) per model; (2) reads per-tenant token counts over the same window from CloudWatch Logs Insights against agent structured logs (carrying `tenant_id`, `model_id`, `input_tokens`, `output_tokens`); (3) computes per-tenant attribution ratio = tenant tokens / total tokens, per tier and per input/output direction; (4) multiplies by CUR-sourced cost; (5) writes to `{env}-tenant-cost-attribution` (PK `tenant_id`, SK `{date_hour}#{model_id}`, one row per tenant × hour × tier). The estimated-cost metrics supply only the proportional breakdown by agent/strategy/quadrant — scaled to sum to the CUR total — not the headline figure.

## Alternatives Considered

- **Estimated cost only (no CUR join).** Rejected: estimated cost uses baked-in constants and misses actual discounts, cross-region premiums, and free-tier offsets; it cannot produce billing-accurate chargeback and is not suitable as an invoice basis.
- **CUR directly attributed by model (no per-tenant split).** Rejected: CUR carries no `tenant_id` dimension; without the per-tenant token share from CloudWatch metrics the CUR total cannot be split across tenants.
- **Real-time per-tenant cost gate using CUR.** Rejected: CUR lags hours behind real time and is unsuitable for admission gating; the real-time spend gate (AD-82) must read the estimated counter (`{env}-tenant-cost-budget`) for timeliness. Two cost sources are a deliberate design, not a mistake.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Billing-accurate per-tenant chargeback: the CUR-authoritative total is preserved, and the per-agent/strategy/quadrant breakdown is scaled to sum exactly to that total | Timeliness: CUR lags hours, so this table is not suitable for real-time gating — the admission spend gate (AD-82) reads the estimated counter instead |
| Decommissioned tenants retain their attribution rows for a 397-day financial-audit TTL, independent of the tenant lifecycle purge (PRD-017) | Two cost sources (estimated for gating, CUR for billing) are deliberate complexity, justified by keeping the invoice exact and the gate timely |

The `agentcore.session_seconds` metric (emitted by the shared invocation wrapper, AD-13) backs per-tenant AgentCore attribution because the AWS-managed `AWS/BedrockAgentCore` namespace carries no `tenant_id` dimension and cannot directly support per-tenant attribution.

## Results

`{env}-tenant-cost-attribution` DynamoDB table (PK `tenant_id`, SK `{date_hour}#{model_id}`; non-key: `date_hour`, `model_id`, `input_cost_usd`, `output_cost_usd`, `total_cost_usd`, `attribution_pct`, `cur_total_usd_window`, `ttl`). The monthly per-tenant cost report (REQ-CST011) reads the headline total from this table and the proportional shape (by agent/strategy/quadrant, cache savings, retry overhead) from §6.1 `token.estimated_cost_usd` metrics scaled to match the CUR total. The table is exempt from the PRD-017 §6.2 tenant purge — financial records age out only via their own 397-day TTL. AD-82 reads the near-real-time estimated counter (`{env}-tenant-cost-budget`, REQ-CST015) for the real-time spend gate; conflating the two sources would make the gate stale or the invoice wrong.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
