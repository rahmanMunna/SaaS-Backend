# ADR-0008: Ports & adapters for payment, mail and storage vendors

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
The system integrates two payment gateways (Stripe, SSLCommerz), two mail transports (SMTP/Mailpit, Resend) and an S3-compatible store (MinIO, whose distribution model changed in 2025). Business logic should not depend on vendor SDKs, and tests should run without real vendors.

## Decision
Define **ports** (TypeScript interfaces) in the application layer — `PaymentProvider`, `MailProvider`, `StorageProvider` — and implement them as **adapters** in the infrastructure layer. The active adapter is chosen by configuration (e.g. `MAIL_DRIVER`) and injected through NestJS providers. Tests use in-memory fakes.

## Alternatives considered
| Option | Pros | Cons | Why not chosen |
|---|---|---|---|
| Call vendor SDKs directly in services | Less code upfront | Vendor lock-in; hard to test; duplicated logic for two gateways | Poor maintainability |

## Consequences
- **Positive:** swapping Resend for SES or MinIO for another S3 server is a single adapter; business logic is unit-testable; demonstrates the Strategy/Adapter patterns.
- **Negative:** an extra abstraction layer; ports must be designed around our needs, not a single vendor's API.
