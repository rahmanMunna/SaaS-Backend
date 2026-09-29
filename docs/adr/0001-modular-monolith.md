# ADR-0001: Modular monolith with separate API and worker processes

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
The system has several concerns (identity, tenancy, business data, files, billing, email) and needs background processing. It is built by one developer, uses one database, and must run with a single `docker compose up` (FR-OPS-01). Maintainability (NFR-08) and independent scaling of request handling vs. background work (NFR-05) matter.

## Decision
Build a **modular monolith**: one NestJS codebase and one Docker image, organized into modules with strict boundaries (one public service per module, cross-module side effects via events, no circular imports). The image runs as **two processes**: `api` (HTTP) and `worker` (queue consumers, outbox relay, scheduled jobs).

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Microservices | Independent deploys and scaling per service | Distributed transactions, network failure modes, service discovery, much higher ops cost | Solves organizational scaling problems we don't have |
| Single process (API does background work) | Simplest | Slow jobs compete with requests; can't scale separately; a crash kills both | Violates NFR-03/05 |

## Consequences
- **Positive:** simple local development and debugging; one transaction can span modules when needed; modules can be extracted later because boundaries are explicit.
- **Negative:** one deployable — a bad release affects everything; boundaries are enforced by convention and review, not by the network.
- **Follow-ups:** consider a lint rule (e.g. dependency-cruiser) to enforce module boundaries automatically.
