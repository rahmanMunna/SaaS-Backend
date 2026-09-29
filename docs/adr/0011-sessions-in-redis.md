# ADR-0011: Refresh-token sessions stored in Redis with rotation

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
Access tokens are short-lived stateless JWTs (15 min). Users need long sessions (7 days) that can be revoked instantly (logout, logout everywhere, password reset, stolen token) — see FR-AUTH-03/04/05.

## Decision
Refresh tokens are opaque random values. Only their **hash** is stored in Redis under `refresh:{sid}` with a 7-day TTL, and each user's session IDs are tracked in `user_sessions:{userId}`. Every refresh **rotates** the token. If a previously rotated token is presented again (**reuse detection**), the whole session is revoked.

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Sessions table in Postgres | Durable; queryable history | Extra DB load on every refresh; expiry needs a cleanup job | Redis TTLs fit the data's lifetime naturally |
| Long-lived JWT refresh token (stateless) | No storage | Cannot be revoked before expiry | Fails revocation requirements |
| Server-side sessions with cookies only | Simple for browser apps | API is also used by non-browser clients | Less flexible |

## Consequences
- **Positive:** instant revocation; automatic expiry; fast lookups; demonstrates Redis TTLs and sets.
- **Negative:** losing Redis data logs everyone out (acceptable; AOF persistence reduces the risk); no long-term session history (audit log covers logins).
