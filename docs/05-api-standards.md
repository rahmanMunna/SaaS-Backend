# API Standards & Endpoint Catalog — TaskHub

| Field | Value |
|---|---|
| Style | REST, JSON over HTTPS |
| Base URL | `http://localhost:3000/api/v1` |
| Interactive docs | `http://localhost:3000/api/docs` (Swagger UI, non-production only) |
| Status | Draft v1.0 · Last updated 2026-09-28 |
| Related | [PRD](01-PRD.md) · [System Flows](03-system-flows.md) · [ADR-0009](adr/0009-api-conventions.md) |

---

## 1. General conventions

| Topic | Standard |
|---|---|
| Versioning | URI prefix `/api/v1`. Breaking changes → `/api/v2`; additive changes (new fields, new endpoints) are non-breaking |
| Resource naming | Plural, kebab-case nouns: `/projects`, `/upload-intent`. Nesting max one level: `/projects/{projectId}/tasks` |
| Actions | Non-CRUD actions are sub-resources with `POST`: `/projects/{id}/archive`, `/files/{id}/complete` |
| JSON fields | `camelCase` in the API (DB stays `snake_case`; mapping happens in DTOs) |
| IDs | UUID strings; invalid UUIDs in the path → 400 |
| Timestamps | ISO 8601 in UTC: `2026-09-28T10:15:30.000Z` |
| Dates | `YYYY-MM-DD` (e.g. `dueDate`) |
| Money | Integer minor units + currency: `{"amountMinor": 900, "currency": "USD"}` |
| Nulls | Optional fields present with `null` rather than omitted, in responses |
| Content type | `application/json; charset=utf-8` (except webhooks/IPN, which follow the provider's format) |

## 2. Authentication & tenancy

- Header: `Authorization: Bearer <accessToken>`.
- The **active tenant** is the `tid` claim inside the token. There is **no** tenant ID in URLs or headers for tenant-scoped endpoints. Change it with `POST /auth/switch-tenant`. → [ADR-0003](adr/0003-active-tenant-in-jwt.md)
- Public endpoints are explicitly marked `@Public()`; everything else requires a token **and** a declared permission (deny by default).

## 3. HTTP methods & status codes

| Method | Use | Success |
|---|---|---|
| `GET` | Read; safe and idempotent | 200 |
| `POST` | Create or trigger an action | 201 (created) · 200 (action with body) · 202 (accepted for async processing) |
| `PATCH` | Partial update | 200 |
| `PUT` | Not used (no full-replacement semantics needed) | – |
| `DELETE` | Delete (soft) | 204 |

| Code | When |
|---|---|
| 400 | Validation error, malformed JSON, invalid UUID |
| 401 | Missing, invalid or expired access token; bad credentials |
| 403 | Authenticated but not allowed: missing permission, suspended tenant/membership, plan limit reached |
| 404 | Resource does not exist **or belongs to another tenant** (never reveal existence) |
| 409 | Conflict with current state: duplicate unique value, idempotency key reused with a different body |
| 410 | Gone: expired or revoked invitation / token |
| 413 | Payload too large |
| 422 | Semantically invalid request that passes validation (e.g. assigning a task to a non-member) |
| 429 | Rate limit exceeded |
| 500 | Unexpected server error (details only in logs, never in the response) |
| 503 | A dependency is unavailable (database, storage) |

## 4. Error format — RFC 9457 Problem Details

All errors use `Content-Type: application/problem+json`:

```json
{
  "type": "https://taskhub.dev/problems/validation-error",
  "title": "Validation failed",
  "status": 400,
  "detail": "The request body contains invalid fields.",
  "instance": "/api/v1/projects",
  "requestId": "01J8Z6Q2K3W9Y1XN5V7B4C2D0E",
  "errors": [
    { "field": "name", "message": "name must be shorter than or equal to 120 characters" }
  ]
}
```

| `type` suffix | Status | Meaning |
|---|---|---|
| `validation-error` | 400 | Field errors in `errors[]` |
| `unauthorized` | 401 | |
| `forbidden` | 403 | Missing permission |
| `tenant-suspended` | 403 | Organization suspended |
| `plan-limit-reached` | 403 | Extra fields: `limit`, `current`, `max` |
| `not-found` | 404 | |
| `conflict` | 409 | |
| `gone` | 410 | |
| `rate-limited` | 429 | `Retry-After` header set |
| `internal-error` | 500 | Generic message only |

Stack traces, SQL errors and internal messages are **never** returned to clients.

## 5. Pagination, filtering & sorting

**Cursor (keyset) pagination** for all lists:

```
GET /api/v1/projects?limit=20&cursor=eyJjIjoiMjAyNi0wOS0yOFQxMDoxNTozMFoiLCJpIjoiMDE5In0
```

```json
{
  "data": [ { "id": "…", "name": "…" } ],
  "meta": { "limit": 20, "nextCursor": "eyJ…", "hasMore": true }
}
```

- `limit`: default 20, max 100.
- `cursor`: opaque base64url string encoding the sort key of the last item (e.g. `createdAt` + `id`). Clients must not parse it.
- Filtering: plain query parameters: `?status=todo&assigneeId=…&priority=high`.
- Search: `?q=term` (case-insensitive substring match).
- Sorting: `?sort=-createdAt` (`-` = descending); only whitelisted fields are allowed.
- Single-resource responses return the object directly (no `data` envelope).

## 6. Idempotency

`POST` endpoints that start payments accept `Idempotency-Key: <uuid>`:
- Same key + same body within 24 h → the **stored original response** is returned; no duplicate side effect.
- Same key + different body → 409.
- Stored in Redis as `idem:{tenantId}:{key}`.

## 7. Rate limiting

| Scope | Limit |
|---|---|
| Global per user (authenticated) | 300 requests / minute |
| Global per IP (anonymous) | 60 requests / minute |
| `POST /auth/login` | 5 / minute per IP **and** per email |
| `POST /auth/password/forgot` | 3 / hour per email |
| `POST /auth/register` | 10 / hour per IP |

Responses include `X-RateLimit-Limit`, `X-RateLimit-Remaining` and, on 429, `Retry-After`.

## 8. Standard headers

| Header | Direction | Purpose |
|---|---|---|
| `Authorization` | request | Bearer access token |
| `X-Request-Id` | both | Correlation ID; generated if absent, always echoed in the response |
| `Idempotency-Key` | request | See §6 |
| `Retry-After` | response | On 429 / 503 |
| `Location` | response | On 201, URL of the created resource |

## 9. Endpoint catalog

`Auth` column: **Public**, **User** (valid token, no tenant permission needed), a **permission key**, or **SuperAdmin**.

### 9.1 Auth & account
| Method | Path | Auth | Description | Req |
|---|---|---|---|---|
| POST | `/auth/register` | Public | Create user + organization, return tokens | FR-AUTH-01 |
| POST | `/auth/login` | Public | Log in | FR-AUTH-02 |
| POST | `/auth/refresh` | Public (refresh token) | Rotate refresh token, new access token | FR-AUTH-03 |
| POST | `/auth/logout` | User | Revoke current session | FR-AUTH-04 |
| POST | `/auth/logout-all` | User | Revoke all sessions | FR-AUTH-04 |
| POST | `/auth/password/forgot` | Public | Request reset email (always 200) | FR-AUTH-05 |
| POST | `/auth/password/reset` | Public | Reset with token | FR-AUTH-05 |
| POST | `/auth/email/verify` | Public | Verify email with token | FR-AUTH-06 |
| GET | `/auth/tenants` | User | List my organizations and roles | FR-AUTH-07 |
| POST | `/auth/switch-tenant` | User | New token for another organization | FR-AUTH-07 |
| GET | `/me` | User | Current user profile + active tenant + permissions | FR-AUTH-08 |
| PATCH | `/me` | User | Update name | FR-AUTH-08 |
| POST | `/me/password` | User | Change password | FR-AUTH-08 |

### 9.2 Organization
| Method | Path | Auth | Description | Req |
|---|---|---|---|---|
| GET | `/tenant` | `tenant:read` | Current organization + plan + usage | FR-TEN-01 |
| PATCH | `/tenant` | `tenant:update` | Update name | FR-TEN-02 |
| POST | `/tenants` | User | Create an additional organization | FR-TEN-03 |
| DELETE | `/tenant` | `tenant:delete` | Delete organization | FR-TEN-04 |

### 9.3 Members, invitations & roles
| Method | Path | Auth | Description | Req |
|---|---|---|---|---|
| GET | `/members` | `member:read` | List members | FR-MEM-04 |
| PATCH | `/members/{userId}` | `member:update-role` | Change role | FR-MEM-05 |
| DELETE | `/members/{userId}` | `member:remove` | Remove member | FR-MEM-06 |
| POST | `/invitations` | `member:invite` | Invite by email | FR-MEM-01 |
| GET | `/invitations` | `member:invite` | List pending invitations | FR-MEM-03 |
| DELETE | `/invitations/{id}` | `member:invite` | Revoke invitation | FR-MEM-03 |
| GET | `/invitations/preview?token=…` | Public | Show organization name/role before accepting | FR-MEM-02 |
| POST | `/invitations/accept` | Public | Accept (creates account if needed) | FR-MEM-02 |
| GET | `/permissions` | `role:read` | Permission catalog | FR-RBAC-03 |
| GET | `/roles` | `role:read` | System + custom roles | FR-RBAC-03 |
| POST | `/roles` | `role:manage` | Create custom role | FR-RBAC-03 |
| PATCH | `/roles/{id}` | `role:manage` | Update custom role | FR-RBAC-03 |
| DELETE | `/roles/{id}` | `role:manage` | Delete custom role (not if assigned) | FR-RBAC-03 |

### 9.4 Projects & tasks
| Method | Path | Auth | Description | Req |
|---|---|---|---|---|
| GET | `/projects` | `project:read` | List (`q`, `status`, cursor) | FR-PRJ-02 |
| POST | `/projects` | `project:create` + plan limit | Create | FR-PRJ-01 |
| GET | `/projects/{id}` | `project:read` | Get | FR-PRJ-03 |
| PATCH | `/projects/{id}` | `project:update` | Update | FR-PRJ-04 |
| POST | `/projects/{id}/archive` | `project:update` | Archive | FR-PRJ-04 |
| POST | `/projects/{id}/unarchive` | `project:update` + plan limit | Unarchive | FR-PRJ-04 |
| DELETE | `/projects/{id}` | `project:delete` | Soft delete | FR-PRJ-05 |
| GET | `/projects/{projectId}/tasks` | `task:read` | List (`status`, `assigneeId`, `priority`, `sort`) | FR-TSK-02 |
| POST | `/projects/{projectId}/tasks` | `task:create` | Create | FR-TSK-01 |
| GET | `/tasks/{id}` | `task:read` | Get | FR-TSK-03 |
| PATCH | `/tasks/{id}` | `task:update` | Update (incl. status, assignee) | FR-TSK-03/04 |
| DELETE | `/tasks/{id}` | `task:delete` | Soft delete | FR-TSK-03 |
| GET | `/tasks/mine` | `task:read` | My tasks across projects | FR-TSK-05 |

### 9.5 Files
| Method | Path | Auth | Description | Req |
|---|---|---|---|---|
| POST | `/tasks/{taskId}/files/upload-intent` | `file:upload` + quota | Create pending file + presigned PUT URL | FR-FILE-01 |
| POST | `/files/{id}/complete` | `file:upload` | Verify and mark ready | FR-FILE-02 |
| POST | `/tasks/{taskId}/files` | `file:upload` + quota | Multipart upload through the API (alternative) | FR-FILE-08 |
| GET | `/tasks/{taskId}/files` | `file:read` | List ready files | FR-FILE-03 |
| GET | `/files/{id}/download` | `file:read` | 302 redirect to presigned GET URL | FR-FILE-04 |
| DELETE | `/files/{id}` | `file:delete` | Delete (object removed async) | FR-FILE-05 |

### 9.6 Billing
| Method | Path | Auth | Description | Req |
|---|---|---|---|---|
| GET | `/plans` | Public | Plans and limits | FR-BILL-01 |
| GET | `/billing/subscription` | `billing:read` | Current subscription + usage vs limits | FR-BILL-02 |
| GET | `/billing/payments` | `billing:read` | Payment history | FR-BILL-08 |
| POST | `/billing/stripe/checkout` | `billing:manage` + Idempotency-Key | Create Checkout Session | FR-BILL-03 |
| POST | `/billing/stripe/portal` | `billing:manage` | Customer Portal URL | FR-BILL-05 |
| POST | `/billing/webhooks/stripe` | Public (signature) | Stripe webhook receiver (raw body) | FR-BILL-04 |
| POST | `/billing/sslcommerz/init` | `billing:manage` + Idempotency-Key | Start SSLCommerz payment | FR-BILL-06 |
| POST | `/billing/sslcommerz/ipn` | Public (validated) | IPN receiver | FR-BILL-06 |
| POST | `/billing/sslcommerz/success` · `/fail` · `/cancel` | Public | Browser redirect targets → redirect to a result page | FR-BILL-06 |

### 9.7 Audit, admin & operations
| Method | Path | Auth | Description | Req |
|---|---|---|---|---|
| GET | `/audit-logs` | `audit:read` | Filter by `actorId`, `action`, `from`, `to` | FR-AUD-02 |
| GET | `/admin/tenants` | SuperAdmin | All organizations with plan & usage | FR-ADM-01 |
| POST | `/admin/tenants/{id}/suspend` | SuperAdmin | Suspend | FR-ADM-02 |
| POST | `/admin/tenants/{id}/reactivate` | SuperAdmin | Reactivate | FR-ADM-02 |
| GET | `/admin/queues` | SuperAdmin | Bull Board UI | FR-ADM-03 |
| GET | `/health/live` | Public | Liveness | FR-OPS-02 |
| GET | `/health/ready` | Public | Readiness (Postgres, Redis, MinIO) | FR-OPS-02 |

## 10. DTO & validation rules

- Separate **request DTOs** (validated with class-validator) and **response DTOs** (explicitly mapped). Entities are **never** returned directly — this prevents leaking columns like `password_hash`.
- Global `ValidationPipe`: `whitelist: true`, `forbidNonWhitelisted: true`, `transform: true`.
- Strings are trimmed; emails lower-cased; lengths match DB `CHECK` constraints.
- Every DTO property is documented for Swagger (type, example, constraints).
