# AD-076 — Make the Master Data Store the Source of Identity Using Deterministic uuid5

**Theme:** Test Tenant Platform & Data
**Catalog:** AD-76 · **Source PRD:** PRD-015 · **Status:** Accepted · **Related:** AD-75, AD-77

## Context

The test-tenant integration must emulate a real ERP: entities (categories, suppliers, items, purchase requisitions) must have stable IDs that survive re-ingestion and cross-dataset resolution so that the PR → Negotiation → Award → PO trace chain remains unbroken. With three public datasets loaded in phases, the same supplier can appear by name in Phase 1 and again in Phase 2; without a pre-existing ID registry, there is no authority to resolve these references to one entity.

## Decision

Make the master data store the source of identity. Mint all entity IDs as `uuid5(NAMESPACE_DNS, f"{tenant_id}:{entity_type}:{name}")`. Buyer Team's `ingest_*` tools preserve master store IDs verbatim as domain entity IDs — they do not re-mint them.

## Alternatives Considered

- **Auto-generated UUIDs (uuid4) at ingest time.** Rejected: non-deterministic; the same supplier loaded twice produces two different IDs, breaking cross-dataset resolution and the trace chain.
- **A centralized ID registry service.** Rejected: adds an external dependency and operational overhead where deterministic derivation achieves the same result without infrastructure.
- **Buyer Team as the source of identity (IDs minted on domain ingest).** Rejected: Buyer Team would need to receive and resolve name-based references before assigning IDs, which defeats the purpose of a canonical master store.

## Trade-offs

| Gained | Given up |
| --- | --- |
| Stable idempotency keys — the same name always yields the same UUID, so re-ingestion upserts rather than duplicates | ID opacity — the UUID is opaque; its derivation from name is a non-obvious convention |
| Cross-dataset entity resolution with no ID registry — a supplier in Phase 1 and Phase 2 resolves to one canonical entity | Rename stability — a genuine rename produces a different UUID, minting a new entity unless explicitly migrated |
| The PR → Negotiation → Award → PO trace chain stays unbroken because IDs are preserved verbatim across the boundary | Names are load-bearing — `name.lower().strip()` normalization mitigates trivial variants but not genuine renames |

## Results

The `uuid5(NAMESPACE_DNS, f"{tenant_id}:{entity_type}:{name}")` formula is shared by the PRD-012 legacy loader and the PRD-015 master store loader. The master store is the read source for Categories, Suppliers, and Items in the Test Tenant App via the `tenant-mdm-emulator` MCP server (AD-77). Buyer Team's integration tool idempotency key is naturally stable: `(tenant_id, source_system="tenant-mdm-emulator", external_id=master_id)`. The preserved ID chain underpins the PR → PO trace that is the demo's capstone, as visualized in the Test Tenant Application (PRD-013).

---
*Part of the [Buyer Team architecture](https://buyer-team.com) decision record · by [Gustavo Peixoto de Azevedo](https://linkedin.com/in/gpazevedo)*
