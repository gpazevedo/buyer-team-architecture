# AD-126 — Daily Cost Explorer Poller Republishes Real AWS-Billed Cost to CloudWatch

**Theme:** Cost Architecture & Optimization
**Catalog:** AD-126 · **Source PRD:** PRD-009 · **Status:** Accepted · **Related:** AD-61, AD-13

## Context

Every cost signal in the platform before this change was a Bedrock-token *estimate* — `negotiation.total_cost_usd`, computed from `resilience/pricing.py`'s baked-in per-token rate constants at emission time (AD-61's correction note). That has no visibility into non-Bedrock AWS spend (Lambda, DynamoDB, Step Functions, SQS, S3) and no ground truth against what AWS actually bills — a rate-table drift or a cross-region inference detour would go unnoticed indefinitely. AD-61 specified a full per-tenant CUR/Athena join as the authoritative fix but that pipeline was never built (confirmed, its own correction note). A smaller, immediately buildable piece of that gap — real, aggregate AWS-billed cost, even without a tenant dimension — was still entirely unaddressed: nothing in the platform polled Cost Explorer at all.

## Decision

New `lambdas/finops_cost_poller` Lambda calls Cost Explorer's `GetCostAndUsage` (hourly granularity, grouped by SERVICE) for the trailing 48h and republishes each hourly bucket as a `procurement/cost` CloudWatch datapoint — real, per-service AWS-billed cost, not a token-based estimate. Scheduled once a day via EventBridge, not hourly. Also exposed as a synchronous on-demand trigger (`test_tenant_app`'s "Poll AWS Costs Now" button → `POST /api/costs/poll` → the same Lambda) for demo/inspection without waiting on the schedule. A new "AWS Cost by Service — Hourly" widget surfaces it on the existing FinOps CloudWatch dashboard.

## Alternatives Considered

- **Poll hourly instead of daily.** Rejected: `GetCostAndUsage` returns every hourly bucket in the requested window in a single $0.01 call, so the trailing-48h window already contains every new bucket once a day — hourly polling would be ~8× more Cost Explorer spend (~$7/mo vs ~$0.30/mo) for identical data.
- **Build AD-61's full per-tenant CUR/Athena join now instead.** Rejected (deferred, not abandoned): per-tenant attribution needs a tenant-dimension bridge — CloudWatch per-tenant token metrics joined against CUR — that is materially more infrastructure than an aggregate service-level poll. This ADR ships the aggregate signal first rather than continuing to have zero real-cost visibility while that larger pipeline stays unbuilt.
- **Rely on the existing Bedrock-token estimate alone.** Rejected: has no visibility into non-Bedrock spend and drifts from actual billing since it's computed from baked-in rate constants, not real AWS invoice data.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Real, per-service AWS-billed cost visible on the FinOps dashboard for the first time, not just a Bedrock-token estimate | Still aggregate (account-wide), not per-tenant — does not by itself deliver AD-61's original per-tenant chargeback goal |
| Daily polling costs ~$0.30/mo vs ~$7/mo hourly for identical data | Cost visibility lags up to 24h behind real spend (and Cost Explorer's own data lags further) — not suitable as a real-time signal, the same class of limitation AD-61 already accepted for its unbuilt CUR pipeline |
| On-demand poll button gives immediate visibility for demo/inspection without waiting on the daily schedule | Each on-demand call incurs the same $0.01 Cost Explorer charge; not rate-limited beyond normal UI interaction |

The poller Lambda has no `vpc_config`, so it keeps running even when the dev NAT Gateway/interface endpoints are released to save cost — each run adds one CloudWatch custom metric per distinct `service_name`, a real contributor toward the account's 10-free-tier-custom-metrics cap. `release_vpc.sh`/`restore_vpc.sh` (PR #209) suspend/resume the poller's EventBridge rule alongside the existing warm-floor/heartbeat toggles so a VPC release doesn't keep creating new custom metrics while dev is deliberately idle.

## Results

Shipped PR #208 (merged 2026-07-13): Lambda (6 tests — pagination, batching, current-hour exclusion, single-call-per-window), dashboard widget, and on-demand endpoint (51 backend contract tests including the new route, manual Playwright click-through of the button). PR #209 (same day) added the suspend/resume wiring in `release_vpc.sh`/`restore_vpc.sh`. Does not build AD-61's per-tenant CUR/Athena join — that gap remains open exactly as AD-61 documents; this poller is a narrower, shipped, aggregate-cost sibling to it, not a replacement.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
