# Architecture Decision Records

An ADR captures **one** significant decision: the context, the choice, the alternatives and the consequences. ADRs are immutable once accepted; to change a decision, write a new ADR that *supersedes* the old one and update the old one's status.

Start new ADRs from [the template](0000-template.md). Number them sequentially.

| ADR | Title | Status |
|---|---|---|
| [0001](0001-modular-monolith.md) | Modular monolith with separate API and worker processes | Accepted |
| [0002](0002-multi-tenancy-shared-schema.md) | Multi-tenancy: shared database, shared schema, `tenant_id` | Accepted |
| [0003](0003-active-tenant-in-jwt.md) | Active tenant carried in the access token | Accepted |
| [0004](0004-typeorm-explicit-migrations.md) | TypeORM with explicit migrations | Accepted |
| [0005](0005-transactional-outbox.md) | Transactional outbox for reliable events | Accepted |
| [0006](0006-presigned-url-uploads.md) | Direct-to-storage uploads with presigned URLs | Accepted |
| [0007](0007-webhooks-as-source-of-truth.md) | Payment webhooks as source of truth, with idempotency | Accepted |
| [0008](0008-ports-and-adapters-for-vendors.md) | Ports & adapters for payment, mail and storage vendors | Accepted |
| [0009](0009-api-conventions.md) | API conventions: URI versioning, RFC 9457 errors, cursor pagination | Accepted |
| [0010](0010-bullmq-for-background-jobs.md) | BullMQ on Redis for background jobs | Accepted |
| [0011](0011-sessions-in-redis.md) | Refresh-token sessions stored in Redis with rotation | Accepted |
