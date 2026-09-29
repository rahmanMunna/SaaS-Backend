# ADR-0003: Active tenant carried in the access token

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
A user can belong to several organizations (FR-AUTH-07). Every tenant-scoped request must know which organization it acts on, and that value must not be forgeable.

## Decision
The access token contains the **active tenant** as the `tid` claim, next to `sub` (user) and `sid` (session). `POST /auth/switch-tenant` issues a new token for another organization after checking membership. The `TenantContextGuard` still verifies on every request (via a Redis cache) that the membership is active and the tenant is not suspended, so removals take effect immediately.

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| `X-Tenant-Id` header on each request | Easy to switch per request | Client-controlled; must re-validate membership for every value; easy to forget | Larger attack surface, more complex contract |
| Tenant in URL (`/tenants/{id}/projects`) | Explicit, RESTful | Long URLs; same validation problem as the header | Same as above |
| Subdomain per tenant (`acme.taskhub.dev`) | Great UX for web apps | Needs wildcard DNS/TLS; not useful for a pure API project | Out of scope |

## Consequences
- **Positive:** tenant cannot be tampered with; simple, uniform API paths; matches how Slack/Linear-style products work.
- **Negative:** switching organizations requires a token exchange; a removed member keeps a valid JWT until expiry → handled by the per-request membership check.
