# ADR-0004: TypeORM with explicit migrations

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
We need an ORM that integrates with NestJS, supports transactions and query building, and — because this is a learning project — makes the underlying SQL and schema evolution visible (learning goal L1).

## Decision
Use **TypeORM** with `synchronize: false` in every environment. Schema changes happen only through migrations stored in the repository, generated from entities **and reviewed**, or hand-written for extensions, indexes, grants and RLS. A separate `data-source.ts` serves the CLI.

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Prisma | Excellent DX and type safety | Own schema language; generated client hides SQL; weaker for RLS and advanced Postgres features | Hides too much for the learning goals |
| Drizzle | SQL-like, lightweight, very type-safe | Smaller NestJS ecosystem; less common in NestJS job listings | Viable; TypeORM chosen for NestJS familiarity |
| Knex / raw SQL | Full control | Lots of boilerplate; no entity mapping | Too low-level for the whole app |
| `synchronize: true` | Zero effort | Can silently drop columns/data; no history | Never acceptable beyond a prototype |

## Consequences
- **Positive:** schema history in Git; reviewable SQL; the same process works in production.
- **Negative:** TypeORM has known quirks (relation loading, generated migration noise) — always read generated SQL.
