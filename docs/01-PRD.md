# Product Requirements Document (PRD) — TaskHub

| Field | Value |
|---|---|
| Product | TaskHub — multi-tenant project & task management API |
| Document owner | Habib |
| Status | Draft v1.0 |
| Last updated | 2026-09-28 |
| Related docs | [Architecture](02-architecture.md) · [System Flows](03-system-flows.md) · [Data Model](04-data-model.md) · [API](05-api-standards.md) · [Roadmap](06-roadmap.md) |

---

## 1. Overview

TaskHub is a B2B SaaS backend where **organizations (tenants)** manage **projects** and **tasks**, collaborate with **team members** under **role-based permissions**, attach **files** to tasks, and pay for a **subscription plan** that unlocks higher limits.

The product domain is intentionally simple. The real deliverable is a **production-grade backend foundation**: multi-tenancy, RBAC, background jobs, object storage, payments, transactional email, migrations and a fully dockerized environment.

## 2. Purpose & problem statement

### 2.1 Why this project exists
This is a **learning project**. Its purpose is to gain real, hands-on backend engineering experience by building every infrastructure concern that a real SaaS product has — and by being able to **inspect and operate each piece from the CLI** (psql, redis-cli, mc, Stripe CLI, docker).

### 2.2 The (simulated) customer problem
Small teams need a simple place to organize work into projects and tasks, share files, and control who can do what — without paying for heavyweight tools. They start free and upgrade when they grow.

## 3. Goals, non-goals & success criteria

### 3.1 Product goals
| ID | Goal |
|---|---|
| G1 | A team can sign up, invite members and manage projects/tasks within minutes. |
| G2 | Each organization's data is **fully isolated** from every other organization. |
| G3 | Owners can upgrade to a paid plan and limits change automatically. |
| G4 | Anyone can clone the repository and run the full system with **one command**. |

### 3.2 Learning goals (the real success criteria)
The project is successful when I can **explain and demonstrate** each of the following:

| ID | I can… | Verified by |
|---|---|---|
| L1 | Design a normalized multi-tenant schema and evolve it with migrations | Writing, running and reverting migrations; explaining `EXPLAIN ANALYZE` output |
| L2 | Explain and enforce tenant isolation at multiple layers | Automated cross-tenant isolation tests pass |
| L3 | Implement JWT auth with refresh-token rotation and reuse detection | Inspecting session keys and TTLs in `redis-cli` |
| L4 | Implement RBAC with permission caching and invalidation | Changing a role and seeing the cache key invalidated |
| L5 | Use Redis for cache, rate limiting, sessions and queues — and know the difference | `MONITOR`, `SCAN`, `TTL`, `INFO` sessions |
| L6 | Build reliable background processing (outbox, retries, backoff, DLQ, cron) | Killing the worker mid-flight and losing zero jobs |
| L7 | Store files in S3-compatible storage with presigned URLs and quotas | `mc ls/stat/du` on tenant prefixes |
| L8 | Integrate a subscription payment provider with webhooks, securely and idempotently | Replaying a Stripe event twice → processed once |
| L9 | Integrate a one-time payment gateway and build renewal logic myself | SSLCommerz sandbox payment + renewal cron |
| L10 | Send transactional email asynchronously | Emails visible in Mailpit, then delivered by Resend |
| L11 | Containerize a multi-service system with healthchecks and volumes | `git clone && docker compose up` works on a clean machine |

### 3.3 Non-goals
- Building a frontend UI (the API is exercised with Swagger / Postman / curl).
- Competing with real task managers on features (no Kanban views, comments, real-time updates…).
- Production deployment to a cloud provider, Kubernetes, autoscaling or multi-region.
- Handling real money — all payments run in **test / sandbox mode** only.

## 4. Users & personas

| Persona | Description | Key needs |
|---|---|---|
| **Organization Owner** | Creates the organization; responsible for billing. | Sign up fast, invite the team, upgrade the plan, full control. |
| **Admin** | Trusted team lead appointed by the owner. | Manage members, projects and tasks; see billing but not pay. |
| **Member** | Regular contributor. | Create/update projects and tasks, upload files. |
| **Viewer** | Stakeholder / client with read-only access. | See progress without changing anything. |
| **Super Admin** | Platform operator (me). Not a tenant member. | See all tenants, suspend abusive tenants, check platform health. |

A single user account can belong to **multiple organizations** with a **different role in each** (like Slack or Notion).

## 5. Scope

### 5.1 In scope (v1)
Authentication · organizations (tenants) · membership & invitations · RBAC with permissions · projects · tasks · file attachments · subscription plans & limits · Stripe subscription billing · SSLCommerz payments · transactional email · audit log · super admin endpoints · health checks · API documentation · dockerized environment with migrations and seeds.

### 5.2 Out of scope (v1)
Social login / SSO · two-factor authentication · email change flow · task comments · real-time notifications (WebSockets) · in-app notifications · full-text search · tax / VAT handling · refunds UI · data export · i18n · frontend application.

## 6. Plans, limits & pricing

| Limit / feature | Free | Pro |
|---|---|---|
| Price (Stripe, test) | $0 | $9.00 / month |
| Price (SSLCommerz, sandbox) | ৳0 | ৳1,000 / month |
| Active projects | 3 | 50 |
| Members (seats, incl. owner) | 3 | 20 |
| Total file storage | 100 MB | 5 GB |
| Max size per file | 10 MB | 50 MB |
| Audit log retention visible | 7 days | 90 days |

Rules:
- Every new organization starts on **Free**, with an active subscription and no payment details.
- Limits are **data-driven** (stored in the `plans` table), never hard-coded.
- **Downgrade rule:** when a paid plan ends, nothing is deleted. Existing data stays readable/editable, but **creating** new resources beyond the Free limits is blocked until the organization is back under the limits or upgrades again.

## 7. Roles & permissions

### 7.1 Permission catalog
| Group | Permissions |
|---|---|
| Organization | `tenant:read`, `tenant:update`, `tenant:delete` |
| Members | `member:read`, `member:invite`, `member:update-role`, `member:remove` |
| Roles | `role:read`, `role:manage` |
| Projects | `project:read`, `project:create`, `project:update`, `project:delete` |
| Tasks | `task:read`, `task:create`, `task:update`, `task:delete` |
| Files | `file:read`, `file:upload`, `file:delete` |
| Billing | `billing:read`, `billing:manage` |
| Audit | `audit:read` |

### 7.2 System roles (seeded, not editable)
| Permission group | Owner | Admin | Member | Viewer |
|---|:-:|:-:|:-:|:-:|
| `tenant:read` | ✅ | ✅ | ✅ | ✅ |
| `tenant:update` | ✅ | ✅ | – | – |
| `tenant:delete` | ✅ | – | – | – |
| `member:read` | ✅ | ✅ | ✅ | ✅ |
| `member:invite`, `member:update-role`, `member:remove` | ✅ | ✅ | – | – |
| `role:read` | ✅ | ✅ | ✅ | – |
| `role:manage` | ✅ | ✅ | – | – |
| `project:read`, `task:read`, `file:read` | ✅ | ✅ | ✅ | ✅ |
| `project:create`, `project:update` | ✅ | ✅ | ✅ | – |
| `project:delete` | ✅ | ✅ | – | – |
| `task:create`, `task:update`, `task:delete` | ✅ | ✅ | ✅ | – |
| `file:upload`, `file:delete` | ✅ | ✅ | ✅ | – |
| `billing:read` | ✅ | ✅ | – | – |
| `billing:manage` | ✅ | – | – | – |
| `audit:read` | ✅ | ✅ | – | – |

Business rules:
- Every organization has **exactly one Owner**. The owner cannot be removed or demoted (ownership transfer is out of scope for v1).
- An Admin cannot grant a role with more permissions than the Admin has (no privilege escalation).
- **Deny by default:** any endpoint without an explicit permission requirement is rejected unless marked public.
- **Super Admin** is a platform-level flag on the user, not a tenant role, and has its own `/admin` endpoints.

## 8. Functional requirements

Priority uses **MoSCoW**: **M**ust, **S**hould, **C**ould.

### 8.1 Authentication & account (AUTH)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-AUTH-01 | Visitor can register with name, email, password and organization name. | M | Creates user, organization, Owner membership and Free subscription **atomically**; returns tokens; a welcome email is queued. Duplicate email → 409. Password ≥ 8 chars. |
| FR-AUTH-02 | User can log in with email and password. | M | Returns a 15-min access token and a 7-day refresh token. Wrong credentials return a generic 401 (no hint which field is wrong). Login is rate-limited. |
| FR-AUTH-03 | User can refresh the session. | M | The refresh token is **rotated** on every use. Reusing an old refresh token revokes the whole session family. |
| FR-AUTH-04 | User can log out from the current device, or from all devices. | M | Refresh tokens are revoked; the access token simply expires. |
| FR-AUTH-05 | User can reset a forgotten password. | M | The request endpoint always returns 200 (no user enumeration). The token is single-use and valid 30 min. Success revokes all sessions. |
| FR-AUTH-06 | User can verify their email address. | S | Verification link emailed at registration; unverified users can still log in (v1). |
| FR-AUTH-07 | User can list their organizations and switch the active one. | M | Switching issues a new access token scoped to the chosen organization, only if the user is an active member. |
| FR-AUTH-08 | User can view and update their profile (name, password). | S | Changing the password requires the current password and revokes other sessions. |

### 8.2 Organizations (TEN)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-TEN-01 | Member can view the current organization. | M | Returns name, slug, plan and usage. |
| FR-TEN-02 | Owner/Admin can update organization name. | M | Slug stays unique platform-wide. |
| FR-TEN-03 | Logged-in user can create an additional organization. | C | The user becomes Owner of the new organization on the Free plan. |
| FR-TEN-04 | Owner can delete the organization. | C | Soft delete; the paid subscription must be cancelled first. |

### 8.3 Members & invitations (MEM)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-MEM-01 | Owner/Admin can invite a person by email with a role. | M | Blocked when the seat limit is reached (counting pending invites). The invite expires after 72 h. Only one pending invite per email per organization. |
| FR-MEM-02 | Invitee can accept an invitation. | M | Existing users get a membership; new users create an account in the same step. An expired or revoked invite → 410. |
| FR-MEM-03 | Owner/Admin can list and revoke pending invitations. | S | |
| FR-MEM-04 | Members can list organization members and their roles. | M | Paginated. |
| FR-MEM-05 | Owner/Admin can change a member's role. | M | The Owner role cannot be assigned or removed; no privilege escalation. The member's permission cache is invalidated immediately. |
| FR-MEM-06 | Owner/Admin can remove a member. | M | Takes effect on the member's next request (not after their token expires). |

### 8.4 Roles & permissions (RBAC)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-RBAC-01 | System roles and permissions are seeded on deploy. | M | Seeding is idempotent. |
| FR-RBAC-02 | Every protected endpoint declares its required permission(s). | M | Missing permission → 403. Undeclared endpoint → rejected (deny by default). |
| FR-RBAC-03 | Owner/Admin can create custom roles from the permission catalog. | C | Custom roles are organization-scoped; names unique per organization. |

### 8.5 Projects (PRJ)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-PRJ-01 | Create a project (name, description). | M | Name unique per organization (case-insensitive). Blocked when the plan's project limit is reached. |
| FR-PRJ-02 | List projects with pagination, search by name and filter by status. | M | Cursor pagination. Only the current organization's projects are ever returned. |
| FR-PRJ-03 | Get a single project. | M | A project of another organization returns **404**, never 403. |
| FR-PRJ-04 | Update a project, or archive/unarchive it. | M | Archived projects do not count toward the limit. |
| FR-PRJ-05 | Delete a project. | M | Soft delete; its tasks are hidden with it. |

### 8.6 Tasks (TSK)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-TSK-01 | Create a task in a project (title, description, priority, due date, assignee). | M | The assignee must be an active member of the same organization. |
| FR-TSK-02 | List tasks of a project, filtered by status, assignee and priority, sorted by due date or created date. | M | Cursor pagination. |
| FR-TSK-03 | Get, update and delete a task. | M | Soft delete. |
| FR-TSK-04 | Change task status (`todo` → `in_progress` → `done`). | M | Any transition allowed in v1. |
| FR-TSK-05 | List "my tasks" across all projects of the organization. | S | |

### 8.7 Files (FILE)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-FILE-01 | Upload a file to a task using a presigned URL. | M | MIME type allow-list (images, PDF, text, office docs). Per-file size and total storage quota enforced **before** issuing the URL. Upload URL valid 5 min. |
| FR-FILE-02 | Confirm an upload. | M | The server verifies the object really exists and uses the **real** size from storage. Usage counter is updated. |
| FR-FILE-03 | List a task's files. | M | Only `ready` files are listed. |
| FR-FILE-04 | Download a file. | M | Returns a presigned download URL valid 60 s. |
| FR-FILE-05 | Delete a file. | M | The object is removed asynchronously and storage usage is decreased. |
| FR-FILE-06 | Generate thumbnails for images in the background. | C | |
| FR-FILE-07 | Clean up abandoned uploads automatically. | S | Files still `pending` after 1 h are removed by a scheduled job. |
| FR-FILE-08 | Upload through the API (multipart), as an alternative learning path. | C | Same validations as FR-FILE-01. |

### 8.8 Billing & subscriptions (BILL)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-BILL-01 | Anyone can list available plans and their limits. | M | Public endpoint. |
| FR-BILL-02 | Owner/Admin can view the current subscription and usage vs. limits. | M | |
| FR-BILL-03 | Owner can upgrade to Pro through Stripe Checkout. | M | The plan changes **only** after the verified webhook, never on the redirect. |
| FR-BILL-04 | The system processes Stripe webhooks. | M | Signature verified; each event processed **exactly once** in effect; out-of-order events handled; response < 1 s (processing is async). |
| FR-BILL-05 | Owner can manage/cancel the Stripe subscription through the Stripe Customer Portal. | S | Cancel takes effect at period end. |
| FR-BILL-06 | Owner can pay for one month of Pro through SSLCommerz. | M | Payment confirmed only through IPN + the validation API; amount, currency and transaction ID must match. |
| FR-BILL-07 | The system reminds and expires SSLCommerz subscriptions. | S | Reminder email 3 days before expiry; downgrade to Free after a 3-day grace period. |
| FR-BILL-08 | Owner/Admin can see payment history. | S | |
| FR-BILL-09 | A failed Stripe renewal puts the subscription in `past_due` and emails the Owner. | M | Pro limits stay during `past_due`; downgrade when Stripe finally cancels. |
| FR-BILL-10 | Payment initiation endpoints are idempotent. | S | Same `Idempotency-Key` → same response, no duplicate checkout session. |

### 8.9 Notifications (NOTIF)
All emails are sent asynchronously and retried on failure.

| ID | Email | Trigger | Pri |
|---|---|---|:-:|
| FR-NOTIF-01 | Welcome | Registration | M |
| FR-NOTIF-02 | Invitation | Member invited | M |
| FR-NOTIF-03 | Password reset | Reset requested | M |
| FR-NOTIF-04 | Email verification | Registration | S |
| FR-NOTIF-05 | Payment receipt | Successful payment | S |
| FR-NOTIF-06 | Payment failed | Stripe renewal failed | S |
| FR-NOTIF-07 | Renewal reminder | SSLCommerz subscription expiring | S |

### 8.10 Audit (AUD)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-AUD-01 | Every write action on organization data is recorded. | S | Actor, action, entity, timestamp, request ID, IP. Append-only. |
| FR-AUD-02 | Owner/Admin can browse the audit log. | S | Filter by actor, action, date range; retention per plan. |

### 8.11 Platform administration (ADM)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-ADM-01 | Super Admin can list and search all organizations with plan and usage. | S | |
| FR-ADM-02 | Super Admin can suspend / reactivate an organization. | S | Members of a suspended organization get 403 on every tenant request. |
| FR-ADM-03 | Super Admin can view the queue dashboard (Bull Board). | S | Protected; not reachable by tenant users. |
| FR-ADM-04 | A Super Admin account is created by the seed. | M | Credentials come from environment variables. |

### 8.12 Operations (OPS)
| ID | Requirement | Pri | Acceptance criteria |
|---|---|:-:|---|
| FR-OPS-01 | `docker compose up` starts the whole system on a clean machine. | M | Migrations and seeds run automatically; all services healthy. |
| FR-OPS-02 | Liveness and readiness health endpoints. | M | Readiness checks Postgres, Redis and MinIO. |
| FR-OPS-03 | Interactive API documentation. | M | Swagger UI at `/api/docs` (non-production only). |
| FR-OPS-04 | Graceful shutdown. | S | In-flight requests and jobs finish before exit. |

## 9. Non-functional requirements

| ID | Category | Requirement |
|---|---|---|
| NFR-01 | Security — isolation | No API call can read or modify another organization's data. Verified by automated tests. |
| NFR-02 | Security — general | Follows OWASP API Security Top 10: argon2id passwords, short-lived tokens, rate limiting, input validation, secure headers, no secrets in code or logs. |
| NFR-03 | Performance | p95 latency < 200 ms for CRUD endpoints on local Docker; slow work (email, file processing, webhooks) is always asynchronous. |
| NFR-04 | Reliability | No email or payment event is lost if Redis, the worker or an external provider is temporarily down. At-least-once processing with idempotent consumers. |
| NFR-05 | Scalability | The API is stateless and can run as multiple instances; all shared state lives in Postgres or Redis. |
| NFR-06 | Observability | Structured JSON logs with request ID, tenant ID and user ID; the request ID is carried into background jobs. |
| NFR-07 | Portability | 12-factor configuration; the same image runs in every environment; configuration validated at start-up. |
| NFR-08 | Maintainability | Modular monolith with enforced module boundaries; linting, formatting and tests in CI. |
| NFR-09 | Testability | Unit tests for guards and services; e2e tests for critical flows (auth, isolation, billing webhooks). |
| NFR-10 | Data integrity | Foreign keys, unique and check constraints enforced in the database, not only in code. Money stored as integers. |

## 10. Assumptions & dependencies

| Type | Item |
|---|---|
| Dependency | Stripe account in **test mode** + Stripe CLI for local webhooks. |
| Dependency | SSLCommerz **sandbox** store credentials. IPN requires a publicly reachable URL (use a tunnel such as ngrok or Cloudflare Tunnel during development). |
| Dependency | Resend account (free tier ≈ 3,000 emails/month, 100/day). Without a verified domain, sending is limited to the account owner's own address. |
| Dependency | Docker Desktop with the WSL2 backend (Windows dev machine). |
| Assumption | A single currency per provider: USD for Stripe, BDT for SSLCommerz. |
| Assumption | Monthly billing interval only. |

## 11. Risks

| Risk | Impact | Mitigation |
|---|---|---|
| MinIO community edition no longer ships prebuilt images or the full admin console. | Setup breaks or behaves differently. | Pin an exact image tag; use the `mc` CLI instead of the web console; storage accessed only through an S3 abstraction so it can be swapped. |
| Resend free-tier limits or domain verification. | Emails not delivered to others. | Mailpit in development; Resend used only for real-send testing. |
| SSLCommerz IPN needs a public URL. | Can't receive IPN locally. | Tunnel tool; plus a manual "validate transaction" fallback endpoint for development. |
| Scope creep. | Project never finishes. | Strict MoSCoW; only **Must** items block a phase. |
| Tenant data leak through a forgotten `WHERE tenant_id`. | Critical security bug. | Central scoping, isolation tests, optional Postgres RLS as a second wall. |

## 12. Release milestones

| Milestone | Contents | Target |
|---|---|---|
| M0 — Foundation | Docker environment, NestJS skeleton, migrations & seeds | End of week 2 |
| M1 — Secure multi-tenant core | Auth, tenancy, RBAC, projects & tasks | End of week 4 |
| M2 — Async & email | Redis cache, rate limiting, outbox, queues, worker, emails | End of week 5 |
| M3 — Files | MinIO storage, presigned uploads, quotas, cleanup | End of week 6 |
| M4 — Monetization | Stripe + SSLCommerz subscriptions, plan limits | End of week 8 |
| M5 — Hardening | Tests, graceful shutdown, docs polish, CI | Week 8+ |

Details per phase: [Implementation Roadmap](06-roadmap.md).

## 13. Open questions

| # | Question | Default until decided |
|---|---|---|
| Q1 | Should Stripe offer a free trial? | No trial in v1. |
| Q2 | Should task comments be added in v2? | Out of scope. |
| Q3 | Should unverified emails be blocked from inviting others? | Not blocked in v1. |

## 14. Glossary

| Term | Meaning |
|---|---|
| Tenant / Organization | A customer account that owns all its data. Used interchangeably. |
| Membership | The link between a user and a tenant, holding the user's role in that tenant. |
| Permission | An atomic allowed action, e.g. `project:create`. |
| Role | A named set of permissions. |
| Plan | A pricing tier (Free, Pro) with limits. |
| Subscription | A tenant's current plan and billing state. |
| Outbox | A database table where events are stored in the same transaction as business data, then published to the queue. |
| Presigned URL | A time-limited URL that lets a client upload or download an object directly from storage. |
| Webhook / IPN | An HTTP call from a payment provider to our server to report an event. |
| Idempotent | Performing an operation multiple times has the same effect as performing it once. |
