# TaskHub — Project Documentation

TaskHub is a multi-tenant SaaS backend (projects & tasks) built as a **hands-on backend engineering practice project**: NestJS, PostgreSQL, Redis, BullMQ, MinIO, Stripe / SSLCommerz and Resend, all running in Docker.

## Documents

| # | Document | Answers the question | Audience |
|---|---|---|---|
| 01 | [Product Requirements (PRD)](01-PRD.md) | **What** are we building and **why**? What is in / out of scope? | Everyone — read first |
| 02 | [Architecture](02-architecture.md) | **How** is the system structured? Which components, which responsibilities? | Engineers |
| 03 | [System Flows](03-system-flows.md) | **How** does each use case move through the system, step by step? | Engineers |
| 04 | [Data Model](04-data-model.md) | What tables exist, what are the constraints, indexes and seeds? | Engineers |
| 05 | [API Standards & Catalog](05-api-standards.md) | What does the API contract look like? Which endpoints exist? | Engineers, API consumers |
| 06 | [Implementation Roadmap](06-roadmap.md) | In **what order** do we build it, and how do we verify each step? | Engineers |
| ADR | [Architecture Decision Records](adr/README.md) | **Why** did we choose X over Y? | Engineers, reviewers |

## Recommended reading order

1. PRD → understand the product and its limits.
2. Architecture §1–§8 → get the big picture.
3. System Flows → Flow 3 (request pipeline) and Flow 9 (job lifecycle) are the backbone of everything.
4. Data Model → before writing the first migration.
5. API Standards → before writing the first controller.
6. Roadmap → pick the current phase and build.

## Documentation conventions (docs-as-code)

- Docs live next to the code and change **in the same pull request** as the code they describe.
- Diagrams are written in **Mermaid** so they are diff-able and render on GitHub.
- A significant or hard-to-reverse technical decision gets an **ADR** (`adr/NNNN-title.md`). ADRs are never edited after acceptance; a new ADR *supersedes* an old one.
- Requirement IDs (e.g. `FR-AUTH-01`) from the PRD are referenced in the roadmap, commit messages and tests, so every feature can be traced back to a requirement.
