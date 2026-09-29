# ADR-0005: Transactional outbox for reliable events

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
Many business actions must trigger asynchronous work (emails, file processing, billing side effects). Writing to Postgres and then enqueuing to Redis is a **dual write** across two systems: a crash or Redis outage between the two loses the event, and enqueuing before commit can emit events for data that is rolled back. NFR-04 requires no lost emails or payment events.

## Decision
Business code writes an `outbox_events` row **in the same database transaction** as the business change. A relay in the worker process polls pending rows with `SELECT … FOR UPDATE SKIP LOCKED`, adds them to BullMQ with **job ID = outbox ID**, and marks them published. Consumers are idempotent.

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Enqueue directly after commit | Simple | Event lost if Redis is down or the process crashes after commit | Violates NFR-04 |
| Enqueue inside the transaction | Simple | Job may run before commit or for a rolled-back change | Incorrect |
| Change Data Capture (Debezium + Kafka) | No polling; scales | Heavy infrastructure | Overkill for this scale |

## Consequences
- **Positive:** events are never lost or phantom; Redis outages only delay processing; a well-known industry pattern.
- **Negative:** extra table and relay; small latency (poll interval ≈ 1 s); at-least-once delivery requires idempotent consumers.
- **Follow-ups:** purge published rows after 7 days.
