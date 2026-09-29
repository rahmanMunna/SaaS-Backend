# ADR-0002: Multi-tenancy — shared database, shared schema, `tenant_id`

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
Every organization's data must be fully isolated (NFR-01), while the platform should support many small tenants cheaply, keep migrations simple and allow cross-tenant platform queries for the super admin (FR-ADM-01).

## Decision
All tenants share one database and one schema. Every tenant-owned table has a `tenant_id NOT NULL` column. Isolation is enforced in layers: signed tenant claim → membership guard → request context → tenant-scoped repositories → composite foreign keys → (stretch) Postgres Row-Level Security → automated isolation tests.

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Schema per tenant | Stronger isolation; per-tenant restore | Migrations run N times; connection/search_path complexity; hard past a few thousand tenants | Operational cost too high for many small tenants |
| Database per tenant | Strongest isolation; noisy-neighbor protection | Very expensive; provisioning automation needed; cross-tenant reporting hard | Suited to few large enterprise tenants only |

## Consequences
- **Positive:** cheapest and simplest model; one migration run; easy platform-wide queries; the standard for most B2B SaaS.
- **Negative:** a missing `tenant_id` filter is a data leak → mitigated by central scoping, composite FKs, RLS and tests. Noisy neighbors share resources.
- **Follow-ups:** all tenant indexes start with `tenant_id`; RLS evaluated in Phase 5.
