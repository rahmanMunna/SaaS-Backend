# System Flows — TaskHub

| Field | Value |
|---|---|
| Status | Draft v1.0 |
| Last updated | 2026-09-28 |
| Related | [PRD](01-PRD.md) · [Architecture](02-architecture.md) · [Data Model](04-data-model.md) |

Each flow describes one use case end to end: participants, steps, failure paths and the engineering lesson it teaches. Requirement IDs link back to the [PRD](01-PRD.md).

| # | Flow | Requirements |
|---|---|---|
| 1 | [Tenant signup](#flow-1--tenant-signup) | FR-AUTH-01, FR-NOTIF-01 |
| 2 | [Login, refresh, switch tenant, logout](#flow-2--session-lifecycle) | FR-AUTH-02/03/04/07 |
| 3 | [Authenticated request lifecycle](#flow-3--authenticated-request-lifecycle) | NFR-01, NFR-02, FR-RBAC-02 |
| 4 | [Invite & accept member](#flow-4--invite-and-accept-a-member) | FR-MEM-01/02 |
| 5 | [CRUD with caching](#flow-5--crud-with-caching) | FR-PRJ-*, FR-TSK-* |
| 6 | [File upload & download](#flow-6--file-upload-and-download) | FR-FILE-* |
| 7 | [Stripe subscription](#flow-7--stripe-subscription) | FR-BILL-03/04/05/09 |
| 8 | [SSLCommerz payment & renewal](#flow-8--sslcommerz-payment-and-renewal) | FR-BILL-06/07 |
| 9 | [Background job lifecycle (outbox → queue → worker)](#flow-9--background-job-lifecycle) | NFR-04 |
| 10 | [Password reset](#flow-10--password-reset) | FR-AUTH-05 |
| 11 | [Role change & permission cache invalidation](#flow-11--role-change-and-cache-invalidation) | FR-MEM-05 |

---

## Flow 1 — Tenant signup

```mermaid
sequenceDiagram
    autonumber
    actor V as Visitor
    participant API as api
    participant DB as Postgres
    participant R as Redis
    participant W as worker

    V->>API: POST /api/v1/auth/register {name, email, password, organizationName}
    API->>API: validate DTO, hash password (argon2id)
    API->>DB: BEGIN
    API->>DB: INSERT users
    API->>DB: INSERT tenants (unique slug)
    API->>DB: INSERT memberships (role = Owner)
    API->>DB: INSERT subscriptions (plan = Free, status = active)
    API->>DB: INSERT outbox_events (user.registered)
    API->>DB: COMMIT
    API->>R: store refresh token hash (TTL 7d)
    API-->>V: 201 {accessToken, refreshToken, tenant}
    W->>DB: relay picks pending outbox rows
    W->>R: add job to email queue
    W->>W: send welcome email
```

**Failure paths**
- Email already exists → `409 Conflict` (checked before the transaction and enforced by a unique constraint).
- Any insert fails → the transaction rolls back; no orphan tenant, no email sent.

**Lesson — the dual-write problem.** Writing to Postgres and then to Redis are two separate systems. If the process crashes between them, one side is lost. The **outbox** row commits atomically with the business data, so the event can never be lost or sent for data that does not exist. → [ADR-0005](adr/0005-transactional-outbox.md)

---

## Flow 2 — Session lifecycle

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant API as api
    participant DB as Postgres
    participant R as Redis

    Note over U,R: Login
    U->>API: POST /api/v1/auth/login {email, password}
    API->>DB: find user, verify argon2id hash
    API->>DB: find default membership (last used tenant)
    API->>R: SET refresh:{sid} = hash(token) EX 7d, SADD user_sessions:{userId} sid
    API-->>U: accessToken (sub, tid, sid - 15 min) + refreshToken

    Note over U,R: Refresh with rotation
    U->>API: POST /api/v1/auth/refresh {refreshToken}
    API->>R: GET refresh:{sid}
    alt hash matches current token
        API->>R: replace with hash(newToken), keep previous hash as "used"
        API-->>U: new accessToken + new refreshToken
    else token was already rotated (reuse)
        API->>R: DEL refresh:{sid} (revoke whole session)
        API-->>U: 401 session revoked
    end

    Note over U,R: Switch tenant
    U->>API: POST /api/v1/auth/switch-tenant {tenantId}
    API->>DB: membership active?
    API-->>U: new accessToken with tid = tenantId

    Note over U,R: Logout
    U->>API: POST /api/v1/auth/logout (or /logout-all)
    API->>R: DEL refresh:{sid} (or every sid in user_sessions:{userId})
    API-->>U: 204
```

**Lessons**
- Access tokens are **stateless** (verified by signature only) and short-lived; refresh tokens are **stateful** so they can be revoked.
- **Rotation + reuse detection:** if an attacker steals a refresh token and both the attacker and the victim use it, the second use is detected and the session is killed.
- Invalid credentials always return the same generic message — no hint whether the email exists.

---

## Flow 3 — Authenticated request lifecycle

The backbone of the whole system. Example: `POST /api/v1/projects`.

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant MW as Middleware
    participant G as Guards
    participant P as ValidationPipe
    participant C as Controller
    participant S as Service
    participant REPO as Repository
    participant R as Redis
    participant DB as Postgres
    participant F as ExceptionFilter

    U->>MW: POST /api/v1/projects + Bearer token
    MW->>MW: assign requestId, helmet, CORS, log
    MW->>G: ThrottlerGuard
    G->>R: INCR rate-limit counter
    G->>G: JwtAuthGuard - verify signature and expiry
    G->>R: TenantContextGuard - GET member:{tid}:{uid}
    R-->>G: miss
    G->>DB: load membership + tenant status
    G->>R: cache membership (10 min)
    G->>G: write tenantId, userId, requestId into CLS
    G->>R: PermissionsGuard - SISMEMBER perm:{tid}:{uid} project:create
    G->>DB: PlanLimitGuard - count active projects vs plan limit
    G->>P: all guards passed
    P->>P: validate + transform body (reject unknown fields)
    P->>C: typed DTO
    C->>S: createProject(dto)
    S->>REPO: save (tenantId taken from CLS)
    REPO->>DB: INSERT projects + audit_logs (one transaction)
    S->>R: INCR cache:{tid}:projects:ver
    S-->>C: project
    C-->>U: 201 Created
    Note over G,F: Any exception at any step goes to the ExceptionFilter,<br/>which returns RFC 9457 Problem Details with the requestId
```

| Step | Failure | Status |
|---|---|---|
| Throttler | too many requests | 429 |
| JWT | missing / invalid / expired token | 401 |
| Tenant context | not a member, membership removed, tenant suspended | 403 |
| Permissions | permission missing or route has no declaration | 403 |
| Plan limit | limit reached | 403 `plan_limit_reached` |
| Validation | invalid body | 400 |
| Service | duplicate name | 409 |

**Lesson — separation of concerns.** Each rule lives in exactly one place in the pipeline. The service contains *only* business logic and is trivially unit-testable.

---

## Flow 4 — Invite and accept a member

```mermaid
sequenceDiagram
    autonumber
    actor A as Admin
    actor I as Invitee
    participant API as api
    participant DB as Postgres
    participant W as worker

    A->>API: POST /api/v1/invitations {email, roleId}
    API->>API: guards - member:invite, seat limit (members + pending invites)
    API->>API: role must not exceed Admin's own permissions
    API->>API: token = random 32 bytes, store only sha256(token)
    API->>DB: INSERT invitations (expires in 72h) + outbox invitation.created
    API-->>A: 201
    W->>I: email with link containing the raw token

    I->>API: POST /api/v1/invitations/accept {token, name?, password?}
    API->>DB: find by sha256(token) - pending and not expired?
    alt user already exists
        API->>DB: INSERT memberships
    else new user
        API->>DB: INSERT users + memberships (one transaction)
    end
    API->>DB: UPDATE invitation status = accepted
    API-->>I: 200 + tokens scoped to the new tenant
```

**Lesson.** Secret tokens are stored **hashed**, like passwords. A leaked database dump contains no usable invitation links.

---

## Flow 5 — CRUD with caching

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant API as api
    participant R as Redis
    participant DB as Postgres

    Note over U,DB: Read (cache-aside)
    U->>API: GET /api/v1/projects?limit=20&cursor=abc
    API->>R: GET cache:{tid}:projects:ver
    API->>R: GET cache:{tid}:projects:v{n}:{hash(query)}
    alt cache hit
        R-->>API: JSON
    else cache miss
        API->>DB: SELECT ... WHERE tenant_id = $1 AND deleted_at IS NULL AND (created_at, id) before cursor ORDER BY created_at DESC, id DESC LIMIT 21
        API->>R: SET key JSON EX 60
    end
    API-->>U: 200 {data, meta: {nextCursor}}

    Note over U,DB: Write (invalidate)
    U->>API: PATCH /api/v1/projects/:id
    API->>DB: UPDATE ... WHERE id = $1 AND tenant_id = $2
    API->>R: INCR cache:{tid}:projects:ver
    API-->>U: 200
```

**Lessons**
- **Versioned keys:** one `INCR` invalidates every cached list for the tenant without scanning keys; old entries simply expire.
- **Cursor pagination** stays fast on deep pages and is stable when rows are inserted; fetching `limit + 1` rows tells you whether there is a next page.
- Every query filters by `tenant_id` *and* `deleted_at IS NULL`.

---

## Flow 6 — File upload and download

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant API as api
    participant DB as Postgres
    participant S3 as MinIO
    participant W as worker

    U->>API: POST /api/v1/tasks/:taskId/files/upload-intent {filename, mimeType, size}
    API->>API: file:upload permission, MIME allow-list, size within plan per-file limit
    API->>DB: storage_used + size within plan quota?
    API->>DB: INSERT files (status = pending, key = tenants/{tid}/tasks/{taskId}/{fileId})
    API->>API: presign PUT (5 min, fixed Content-Type)
    API-->>U: {fileId, uploadUrl}
    U->>S3: PUT uploadUrl (file bytes go directly to storage)
    U->>API: POST /api/v1/files/:fileId/complete
    API->>S3: HEAD object - exists? real size?
    API->>DB: BEGIN - status = ready, size = real size, storage_used += size, outbox file.uploaded - COMMIT
    API-->>U: 200 file metadata
    W->>S3: (images only) create thumbnail

    U->>API: GET /api/v1/files/:fileId/download
    API->>API: file:read, file belongs to tenant
    API-->>U: 302 to presigned GET URL (60 s)
```

**Failure paths**
- Quota or size exceeded → rejected **before** any upload happens.
- Client never calls `complete` → hourly cleanup deletes the `pending` row and object.
- Real size larger than declared → the file is rejected and the object deleted.

**Lessons.** The API never carries file bytes, so it stays small and fast. Never trust client-declared metadata — verify it against storage.

---

## Flow 7 — Stripe subscription

```mermaid
sequenceDiagram
    autonumber
    actor O as Owner
    participant API as api
    participant ST as Stripe
    participant DB as Postgres
    participant W as worker

    O->>API: POST /api/v1/billing/stripe/checkout {planCode: pro} + Idempotency-Key
    API->>API: billing:manage permission
    API->>ST: get or create Customer (store stripe_customer_id)
    API->>ST: create Checkout Session (mode subscription, metadata tenantId)
    API-->>O: {checkoutUrl}
    O->>ST: pays with a test card
    ST-->>O: redirect to success URL (display only, NOT proof of payment)

    ST->>API: POST /api/v1/billing/webhooks/stripe (raw body + Stripe-Signature)
    API->>API: verify signature with webhook secret
    API->>DB: INSERT webhook_events (provider, event_id) - unique
    alt duplicate event
        API-->>ST: 200 (already received)
    else new event
        API->>W: enqueue billing job
        API-->>ST: 200 immediately
        W->>ST: retrieve current subscription state
        W->>DB: upsert subscriptions (plan Pro, status active, period end), INSERT payments, outbox payment.succeeded
        W->>DB: mark webhook_event processed
    end
```

| Stripe event | Effect |
|---|---|
| `checkout.session.completed` | Link Stripe subscription ID to the tenant; plan = Pro, status = active |
| `invoice.paid` | Extend `current_period_end`, record payment, send receipt |
| `invoice.payment_failed` | status = `past_due`, email the Owner |
| `customer.subscription.updated` | Sync status, `cancel_at_period_end`, period dates |
| `customer.subscription.deleted` | Back to Free plan (downgrade rule applies) |

**Lessons**
- The webhook is the **source of truth**; the redirect can be faked or never happen.
- Webhooks arrive **more than once** and **out of order** → idempotency table + fetch-current-state-and-upsert, instead of trusting event order.
- Respond fast (< 1 s) and process asynchronously, or the provider retries and creates duplicates.
- Signature verification needs the **raw request body**, so body parsing must be disabled for this route.

Local testing: `stripe listen --forward-to localhost:3000/api/v1/billing/webhooks/stripe` and `stripe trigger invoice.payment_failed`.

---

## Flow 8 — SSLCommerz payment and renewal

```mermaid
sequenceDiagram
    autonumber
    actor O as Owner
    participant API as api
    participant SC as SSLCommerz
    participant DB as Postgres
    participant W as worker

    O->>API: POST /api/v1/billing/sslcommerz/init {planCode: pro}
    API->>DB: INSERT payments (status pending, tran_id, amount 100000, BDT)
    API->>SC: create session (tran_id, amount, success/fail/cancel/ipn URLs)
    SC-->>API: GatewayPageURL
    API-->>O: {gatewayUrl}
    O->>SC: pays in sandbox
    SC->>API: POST IPN {tran_id, val_id, status}
    API->>SC: Validation API (val_id)
    SC-->>API: VALID + amount + currency
    API->>API: compare amount, currency, tran_id with the pending payment
    API->>DB: payment = paid, subscription Pro, period_end = now + 30 days, outbox payment.succeeded
    API-->>SC: 200
    SC-->>O: browser redirect to success URL (display only)

    Note over W,DB: Daily renewal scan (02:00)
    W->>DB: SSLCommerz subscriptions ending within 3 days
    W->>O: renewal reminder email with a payment link
    W->>DB: past period end + 3 days grace - downgrade to Free
```

**Lesson.** A gateway without native recurring billing means **you** own the billing cycle: period tracking, reminders, grace period and downgrade — all driven by scheduled jobs. Never trust the IPN body alone; always confirm with the validation API and compare amounts.

---

## Flow 9 — Background job lifecycle

```mermaid
flowchart LR
    A["Business transaction<br/>writes outbox_events row"] --> B["Relay in worker<br/>SELECT ... FOR UPDATE SKIP LOCKED"]
    B --> C["BullMQ add<br/>jobId = outbox id"]
    C --> D["Outbox row<br/>status = published"]
    C --> E{"Worker processes job"}
    E -- success --> F["completed<br/>(auto-removed later)"]
    E -- error --> G{"attempts left?"}
    G -- yes --> H["delayed<br/>exponential backoff"]
    H --> E
    G -- no --> I["failed set<br/>(dead letter)"]
    I -- "manual retry via Bull Board" --> E
```

| Guarantee | How |
|---|---|
| Event never lost | Stored in Postgres first (outbox) |
| Event never published twice by concurrent relays | `FOR UPDATE SKIP LOCKED` |
| Job never enqueued twice | BullMQ job ID = outbox ID |
| Side effect safe if the job runs twice | Idempotent consumers (e.g. check `sent_at`, unique constraints) |
| Transient errors recover | Retries with exponential backoff |
| Permanent errors are visible | Failed set + logs + Bull Board |

**Lesson.** Distributed systems give you **at-least-once** delivery. "Exactly once" is achieved by making the *effect* idempotent.

---

## Flow 10 — Password reset

```mermaid
sequenceDiagram
    autonumber
    actor U as User
    participant API as api
    participant R as Redis
    participant DB as Postgres
    participant W as worker

    U->>API: POST /api/v1/auth/password/forgot {email}
    API->>DB: find user by email
    opt user exists
        API->>R: SET pwreset:{sha256(token)} = userId EX 1800
        API->>DB: INSERT outbox password_reset.requested
    end
    API-->>U: 200 always (no user enumeration)
    W->>U: email with reset link (raw token)

    U->>API: POST /api/v1/auth/password/reset {token, newPassword}
    API->>R: GETDEL pwreset:{sha256(token)} (single use)
    API->>DB: update password hash
    API->>R: revoke all sessions of the user
    API-->>U: 204
```

**Lessons.** Same response whether or not the email exists; rate-limit per IP and per email; `GETDEL` makes the token single-use atomically.

---

## Flow 11 — Role change and cache invalidation

```mermaid
sequenceDiagram
    autonumber
    actor A as Admin
    participant API as api
    participant DB as Postgres
    participant R as Redis

    A->>API: PATCH /api/v1/members/:userId {roleId}
    API->>API: member:update-role, target is not Owner, no privilege escalation
    API->>DB: UPDATE memberships SET role_id + audit log
    API->>R: DEL perm:{tid}:{userId} and member:{tid}:{userId}
    API-->>A: 200
    Note over API,R: The member's next request misses the cache,<br/>loads the new permissions from Postgres and re-caches them.
```

**Lesson.** Caches need an **explicit invalidation path** for security-relevant data. The TTL is only a safety net, not the mechanism.
