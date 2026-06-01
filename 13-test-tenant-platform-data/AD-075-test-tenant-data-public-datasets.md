# AD-075 — Source Test-Tenant Data from Three Public Datasets; Extract Golden Sets from the Same Source

**Theme:** Test Tenant Platform & Data
**Catalog:** AD-75 · **Source PRD:** PRD-012 · **Status:** Accepted · **Related:** AD-76, AD-77, AD-90

## Context

The system needs a demonstrable, reproducible dataset that exercises all four Kraljic quadrants end-to-end, plus golden evaluation datasets for automated evals (AD-90) — without access to real customer procurement data. No licensing or privacy framework exists for live customer data in a demo or CI environment, and synthetic data risks misrepresenting the Kraljic classification distribution.

## Decision

Source the test tenant's data from three public datasets — Kaggle Kraljic Strategy (~1k rows), Kaggle Procurement KPI (~5k rows), and UCI Online Retail (~541k rows) — loaded in strict phase order (1 → 2 → 3) via the Test Tenant Skill (PRD-012). Extract golden evaluation subsets from the same source data: 200 Kraljic records (50 per quadrant, stratified), 500 KPI records (ranked), and 5,000 UCI records (category-coverage sampled).

## Alternatives Considered

- **Real customer procurement data.** Rejected: licensing and privacy constraints make it unavailable in demo and CI contexts.
- **Fully synthetic data.** Rejected: synthetic data cannot faithfully reproduce the Kraljic distribution or supplier performance statistics needed for accurate evaluation baselines.
- **Status quo / no dedicated test dataset.** Rejected: without a reproducible shared dataset, CI golden evals and demo scenarios cannot be consistently reproduced across environments.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Single, public, reproducible source — no licensing or privacy concerns | Realism: public datasets approximate but do not match real enterprise procurement |
| Golden sets guaranteed consistent with loaded demo data (same source) | Mild train-on-test risk: eval sets derived from the same corpus used in the demo |
| All four Kraljic quadrants and the full supplier lifecycle are covered | Strict phase ordering (1 → 2 → 3) creates cross-dataset dependencies; out-of-order load is an error |

The train-on-test risk is mitigated by stratified and random sampling within phases and by synthetic edge-case generation. Phase coupling is load-bearing: Phase 2 enriches the Supplier skeletons from Phase 1; Phase 3 keyword-matches Items to Phase 1 Categories.

## Results

Implemented in `skills/test_tenant/` (PRD-012). The three load tools (`load_kraljic_dataset`, `load_procurement_kpi_dataset`, `load_uci_retail_dataset`) and `extract_golden_datasets` realize this decision. Post-phase quality assertions gate each load (e.g., ≥95% valid quadrant assignment in Phase 1). Golden files are written to S3 with version tags and consumed by Strands Evals on every PR (AD-90).

This loader is the *legacy* direct-to-domain path. The v1.0 target routes test-tenant ingestion through the master data store via the `tenant-mdm-emulator` MCP server (AD-76, AD-77). Both paths consume the same CSVs and produce the same canonical entities; the legacy path runs in parallel for shadow validation until its retirement is decided.

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
