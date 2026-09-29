# ADR-0007: Payment webhooks as source of truth, with idempotency

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
After paying, the browser is redirected back to our success URL, but that redirect can be skipped, repeated or forged. Payment providers report the real outcome through webhooks (Stripe) or IPN (SSLCommerz), which may be delivered **more than once** and **out of order**, and are retried if we respond slowly (FR-BILL-04/06).

## Decision
- Subscription and payment state change **only** from verified provider notifications, never from redirects.
- Verification: Stripe signature over the raw request body; SSLCommerz validation API with amount, currency and transaction ID checks.
- Every notification is inserted into `webhook_events` with a **unique `(provider, event_id)`**; duplicates are acknowledged and ignored.
- The endpoint responds 200 quickly and hands processing to the `billing` queue.
- Handlers fetch the **current** state from the provider and upsert, rather than relying on event order.

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Trust the success redirect | Simple | Forgeable; lost if the user closes the tab | Insecure and unreliable |
| Process webhooks synchronously | Simpler code | Slow responses trigger provider retries and duplicates | Less reliable |

## Consequences
- **Positive:** correct under retries, duplicates, reordering and user drop-off; full audit trail of provider events.
- **Negative:** short delay between payment and plan change (UI must show "processing"); extra table and queue.
