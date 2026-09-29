# ADR-0009: API conventions — URI versioning, RFC 9457 errors, cursor pagination

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
A consistent, predictable API contract is needed before the first controller is written, so that every module behaves the same for clients and errors are machine-readable.

## Decision
- **Versioning:** URI prefix `/api/v1`.
- **Errors:** RFC 9457 Problem Details (`application/problem+json`) with `type`, `title`, `status`, `detail`, `instance`, `requestId` and optional field `errors[]`.
- **Pagination:** cursor (keyset) based with an opaque cursor, `limit` ≤ 100.
- **Foreign tenant resources:** 404, never 403.
- **Idempotency:** `Idempotency-Key` header on payment-initiating POSTs.

Full details: [API Standards](../05-api-standards.md).

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Header/media-type versioning | Clean URLs | Harder to test with browser/curl; less visible | URI versioning is simpler and most common |
| Custom error envelope | Freedom | Every client learns a bespoke format | A standard exists |
| Offset pagination | Simple, supports "jump to page" | Slow on deep pages; duplicates/skips when data changes | Keyset is stable and fast |
| GraphQL | Flexible queries | Different learning track; caching and authorization per field are harder | Out of scope |

## Consequences
- **Positive:** uniform, standard, documented contract; stable pagination performance.
- **Negative:** cursor pagination cannot jump to an arbitrary page number.
