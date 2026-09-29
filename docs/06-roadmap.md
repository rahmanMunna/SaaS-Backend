# Implementation Roadmap — TaskHub

| Field | Value |
|---|---|
| Duration | ~8 weeks at 2–3 hours/day |
| Status | Draft v1.0 · Last updated 2026-09-28 |
| Related | [PRD §12 Milestones](01-PRD.md#12-release-milestones) · [Architecture](02-architecture.md) |

## How to use this roadmap

- Work through the phases **in order**; each builds on the previous one.
- Tick the checkboxes as you go (they render as checklists on GitHub).
- A phase is **done** only when its **"Prove it from the CLI"** checkpoint passes and its Definition of Done is met.
- Only **Must** requirements block a phase. Should/Could items can be revisited later.
- Workflow per phase: *learn the concept → build → self-review → mentor review → prove it → commit*.

### Definition of Done (every phase)
- [ ] Code builds, lints and is formatted
- [ ] New config values added to `.env.example` and validated at start-up
- [ ] Schema changes delivered as migrations (never `synchronize`)
- [ ] Unit/e2e tests for the new behaviour where it matters (guards, services, flows)
- [ ] Swagger updated for new endpoints
- [ ] Docs updated if the design changed (new ADR if a decision changed)
- [ ] `docker compose down -v && docker compose up` still works from scratch
- [ ] Small, meaningful commits referencing requirement IDs

---

## Milestone M0 — Foundation (weeks 1–2)

### Phase 0 — Tooling & Docker fundamentals · Day 1
- [ ] Install Docker Desktop (WSL2 backend), Node.js 24 LTS, pnpm, Nest CLI, Git, Stripe CLI; optional: DBeaver, RedisInsight
- [ ] Learn: image vs container, layers, volumes (named vs bind), networks, port publishing, `docker compose` lifecycle
- **Prove it:** run a throwaway Postgres container, create a table, delete the container, and explain what happened to the data with and without a volume.

### Phase 1 — Infrastructure with Compose (no app yet) · Days 2–3
- [ ] `docker-compose.yml` with `postgres`, `redis`, `minio`, `minio-init`, `mailpit`
- [ ] Named volumes, healthchecks, `depends_on: condition: service_healthy / service_completed_successfully`
- [ ] Networks `backend` and `edge`
- [ ] `.env` (git-ignored) and `.env.example`
- [ ] Redis with AOF persistence and `noeviction`
- [ ] Pin the MinIO image tag
- **Prove it:**
  - `psql`: `\l`, `\dn`, `\du`
  - `redis-cli`: `PING`, `SET k v EX 30`, `TTL k`, `INFO persistence`, `CONFIG GET maxmemory-policy`
  - `mc alias set`, `mc ls`, `mc cp`, `mc stat`
  - Send a test email to Mailpit via SMTP and see it at `localhost:8025`
  - `down` → `up`: data persists · `down -v`: data gone

### Phase 2 — NestJS skeleton, dockerized · Days 4–6 · FR-OPS-01/02/03
- [ ] `nest new`, strict TypeScript, ESLint + Prettier
- [ ] Folder structure from [Architecture §5.5](02-architecture.md#55-source-layout)
- [ ] Config module with schema validation (fail fast)
- [ ] Multi-stage Dockerfile (deps → build → runtime, non-root user) + `.dockerignore`
- [ ] `docker-compose.override.yml` for development: bind mount, hot reload, `node_modules` volume
- [ ] Global: `ValidationPipe`, RFC 9457 exception filter, request-ID middleware, pino logger, helmet, CORS
- [ ] API prefix `/api/v1`, Swagger at `/api/docs`
- [ ] Health endpoints `/health/live`, `/health/ready` (Postgres, Redis, MinIO)
- **Prove it:** `docker compose up` → `/health/ready` is green; `docker compose stop redis` → readiness turns red and the `api` container becomes unhealthy; logs are JSON with `requestId`.

### Phase 3 — Schema, migrations & seeds · Days 7–11 · FR-RBAC-01, FR-ADM-04
- [ ] TypeORM `DataSource` for the CLI; `synchronize: false`
- [ ] Entities for all tables in [Data Model §3](04-data-model.md#3-table-definitions)
- [ ] Migrations 1–7 from [Data Model §5](04-data-model.md#5-migration-strategy) — generated **and** hand-written ones
- [ ] Least-privilege app DB role + grants (append-only audit log)
- [ ] Idempotent seeds: permissions → system roles → role permissions → plans → super admin → demo tenant (dev only)
- [ ] Container entrypoint: run migrations + seeds, then start
- **Prove it:**
  - `SELECT * FROM migrations;` — explain how TypeORM knows what ran
  - `migration:revert` then `migration:run`
  - `\d+ tasks` — point out the composite FK and partial indexes
  - Try to insert a task referencing another tenant's project → constraint error
  - `EXPLAIN ANALYZE` a tenant-scoped list query with and without its index
  - Run seeds twice → no duplicates

## Milestone M1 — Secure multi-tenant core (weeks 3–4)

### Phase 4 — Authentication · Days 12–15 · FR-AUTH-01…05, 07, 08
- [ ] Register (single transaction), login, argon2id
- [ ] Access JWT (`sub`, `tid`, `sid`, `sa`), refresh token rotation + reuse detection in Redis
- [ ] Logout, logout-all, switch tenant, `/me`
- [ ] Password forgot/reset (email sending stubbed until Phase 9)
- [ ] Global `JwtAuthGuard` + `@Public()`
- **Prove it:** log in, then in `redis-cli` find `refresh:{sid}` and `user_sessions:{userId}` and check their `TTL`; reuse an old refresh token → session revoked; logout → keys gone.

### Phase 5 — Tenant isolation · Days 16–18 · NFR-01
- [ ] Request context with CLS (`tenantId`, `userId`, `requestId`)
- [ ] `TenantContextGuard` (membership + tenant status, cached in Redis)
- [ ] Tenant-aware base repository pattern
- [ ] e2e isolation test suite (tenant A vs tenant B → 404 everywhere)
- [ ] *Stretch:* Postgres RLS ([Data Model §4](04-data-model.md#4-row-level-security-stretch-goal))
- **Prove it:** remove a member in psql, clear their `member:*` key, and show their next request fails with 403 even though their JWT is still valid.

### Phase 6 — RBAC & members · Days 19–22 · FR-MEM-*, FR-RBAC-02/03
- [ ] `@RequirePermissions()` + `PermissionsGuard`, deny by default
- [ ] Permission cache `perm:{tid}:{uid}` + invalidation on change
- [ ] Members: list, change role, remove (owner protection, no privilege escalation)
- [ ] Invitations: create (seat limit), list, revoke, preview, accept
- [ ] *Could:* custom roles
- [ ] Super admin guard + `/admin/tenants` endpoints
- **Prove it:** as a Member, call `DELETE /projects/{id}` → 403; promote to Admin → the `perm:*` key is deleted → the same call now succeeds without logging in again.

### Phase 7 — Projects & tasks CRUD · Days 23–26 · FR-PRJ-*, FR-TSK-*
- [ ] DTOs + response mapping, Swagger
- [ ] Cursor pagination, filters, search (`pg_trgm`), sorting whitelist
- [ ] Soft delete, archive/unarchive
- [ ] `PlanLimitGuard` for projects
- [ ] Audit log entries on every write
- **Prove it:** enable SQL logging and explain every query an endpoint runs; find and fix one N+1; show that the 4th project on the Free plan is rejected.

## Milestone M2 — Async & email (week 5)

### Phase 8 — Redis in depth · Days 27–28 · NFR-03
- [ ] Cache-aside with versioned keys for project/task lists
- [ ] Throttler with Redis storage; per-route limits from [API §7](05-api-standards.md#7-rate-limiting)
- [ ] Idempotency-key interceptor (used later by billing)
- **Prove it:** `redis-cli MONITOR` while calling endpoints; `SCAN 0 MATCH cache:* COUNT 100`; show a cache hit, then a write, then the `ver` key incremented; explain why `KEYS *` is dangerous in production.

### Phase 9 — Outbox, queues, worker & email · Days 29–34 · FR-NOTIF-*, NFR-04
- [ ] `outbox_events` writer used inside business transactions
- [ ] `worker.ts` entry point + `worker` service in compose (same image)
- [ ] Outbox relay (`FOR UPDATE SKIP LOCKED`, job ID = outbox ID)
- [ ] BullMQ queues from [Architecture §11.2](02-architecture.md#112-queues) with retries/backoff
- [ ] Bull Board at `/admin/queues`
- [ ] `MailProvider` port: SMTP (Mailpit) + Resend adapters; templates for all NOTIF emails
- [ ] Repeatable jobs: outbox purge
- [ ] Graceful shutdown for the worker
- **Prove it:**
  - Stop the worker, register 5 users, see pending rows in `outbox_events` and jobs waiting in Redis (`LLEN`/`ZCARD` on `bull:email:*`)
  - Start the worker → all emails appear in Mailpit
  - Force the mail provider to fail → watch retries with backoff, then the job lands in the failed set; retry it from Bull Board
  - Stop Redis, register a user, start Redis → the email is still delivered (outbox)
  - Switch `MAIL_DRIVER=resend` and receive a real email

## Milestone M3 — Files (week 6)

### Phase 10 — File management with MinIO · Days 35–40 · FR-FILE-*
- [ ] `StorageProvider` port + S3 adapter; internal vs public endpoint for presigning
- [ ] Upload intent → presigned PUT → complete (HEAD verification)
- [ ] MIME allow-list, per-file size limit, storage quota with atomic counter
- [ ] Presigned GET download (302)
- [ ] Delete → async object removal; hourly cleanup of pending uploads
- [ ] *Could:* thumbnail job with `sharp`; multipart upload through the API
- **Prove it:** `mc ls --recursive local/taskhub-files/tenants/`, `mc stat` an object, `mc du` a tenant prefix and compare with `tenants.storage_used_bytes`; wait for a download URL to expire and show the error; try to exceed the Free quota.

## Milestone M4 — Monetization (weeks 7–8)

### Phase 11 — Stripe subscriptions · Days 41–47 · FR-BILL-01…05, 09, 10
- [ ] Stripe test product/price; `stripe_price_id` in the plans seed
- [ ] `PaymentProvider` port + Stripe adapter
- [ ] Checkout Session (idempotent), Customer Portal
- [ ] Webhook endpoint: raw body, signature check, `webhook_events` insert, enqueue, 200
- [ ] Billing worker: handle the five events from [Flow 7](03-system-flows.md#flow-7--stripe-subscription); fetch-state-and-upsert
- [ ] Plan limits switch automatically; downgrade rule
- **Prove it:** `stripe listen --forward-to …`; pay with `4242 4242 4242 4242` → plan becomes Pro only after the webhook; `stripe events resend <evt_id>` → processed once (check `webhook_events`); `stripe trigger invoice.payment_failed` → `past_due` + email.

### Phase 12 — SSLCommerz & renewals · Days 48–52 · FR-BILL-06/07/08
- [ ] SSLCommerz adapter behind the same port
- [ ] Init session, IPN handler, validation API, amount/currency/tran_id checks
- [ ] Tunnel (ngrok / Cloudflare Tunnel) for IPN in development
- [ ] Daily renewal scan: reminders, grace, downgrade
- [ ] Payment history endpoint
- **Prove it:** complete a sandbox payment and trace it through `payments`, `webhook_events` and `subscriptions` in psql; manually move `current_period_end` into the past and run the renewal job → grace → Free.

## Milestone M5 — Hardening (week 8+)

### Phase 13 — Quality & polish · Days 53–56 · NFR-06/08/09
- [ ] Unit tests: guards, permission resolution, subscription state transitions
- [ ] e2e: auth, isolation, invitations, billing webhooks (with fixtures)
- [ ] Graceful shutdown for the API; container resource limits
- [ ] Log redaction verified; secrets never logged
- [ ] Root README: prerequisites, `cp .env.example .env`, `docker compose up`, default URLs and credentials, CLI cheat sheet
- [ ] *Optional:* GitHub Actions — lint, test, build image
- [ ] *Optional:* OpenTelemetry tracing, Prometheus metrics
- **Prove it:** a fresh clone on another machine (or a clean WSL distro) reaches a fully working system with one command.

---

## Progress tracker

| Phase | Name | Status |
|---|---|---|
| 0 | Tooling | ⬜ Not started |
| 1 | Infrastructure | ⬜ |
| 2 | NestJS skeleton | ⬜ |
| 3 | Schema, migrations, seeds | ⬜ |
| 4 | Authentication | ⬜ |
| 5 | Tenant isolation | ⬜ |
| 6 | RBAC & members | ⬜ |
| 7 | Projects & tasks | ⬜ |
| 8 | Redis in depth | ⬜ |
| 9 | Outbox, queues, email | ⬜ |
| 10 | Files | ⬜ |
| 11 | Stripe | ⬜ |
| 12 | SSLCommerz | ⬜ |
| 13 | Hardening | ⬜ |

Legend: ⬜ not started · 🟨 in progress · ✅ done
