# ADR-0010: BullMQ on Redis for background jobs

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
Email sending, file processing, webhook processing and scheduled tasks (renewals, cleanups) must run outside the request cycle with retries, backoff, delayed and repeatable jobs, and visibility into failures (NFR-03/04). Redis is already part of the stack.

## Decision
Use **BullMQ** (with `@nestjs/bullmq`) on the existing Redis. Queues: `email`, `files`, `billing`, `audit`. Consumers run only in the `worker` process. Failed jobs remain in the failed set for inspection; Bull Board provides a UI for super admins.

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| RabbitMQ | Mature broker; routing patterns | Another service to run; delayed/repeatable jobs need plugins | Extra infrastructure without clear benefit here |
| Kafka | Event streaming, replay, huge throughput | Heavy; a log, not a job queue | Wrong tool for job processing |
| pg-boss (Postgres-based queue) | No Redis needed; transactional enqueue | Less common in the NestJS ecosystem; learning goal includes Redis queues | Viable alternative, noted for future |
| Cloud queues (SQS) | Managed | Not local-first; costs | Must run fully in Docker |

## Consequences
- **Positive:** first-class NestJS integration; retries, backoff, priorities, delayed and cron jobs built in; inspectable in `redis-cli`.
- **Negative:** Redis must not evict queue keys (`noeviction`) and should persist (AOF); delivery is at-least-once → idempotent consumers required.
