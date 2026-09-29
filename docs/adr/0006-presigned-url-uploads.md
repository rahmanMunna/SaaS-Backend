# ADR-0006: Direct-to-storage uploads with presigned URLs

- **Status:** Accepted
- **Date:** 2026-09-28

## Context
Users attach files up to 50 MB to tasks (FR-FILE-*). Streaming file bytes through the API consumes memory, bandwidth and connection time, and limits API scalability (NFR-03/05).

## Decision
Use a **two-phase upload**: the API validates the request (permission, MIME type, size, quota), creates a `pending` file record and returns a short-lived **presigned PUT URL**. The client uploads directly to MinIO, then calls `complete`; the API verifies the object with `HEAD` (existence and real size) before marking it `ready`. Downloads use short-lived presigned GET URLs. The bucket stays private and object keys are server-generated. A multipart upload through the API is kept as a secondary learning path.

## Alternatives considered
| Option | Pros | Cons | Why not chosen as primary |
|---|---|---|---|
| Proxy upload through the API | Simple; the server sees every byte (easy scanning) | API bandwidth and memory bottleneck; timeouts on large files | Doesn't scale |
| Public bucket | Trivial downloads | Anyone with a URL reads forever; no access control | Insecure |

## Consequences
- **Positive:** API stays lightweight; storage handles the heavy transfer; access remains controlled and time-limited.
- **Negative:** more complex client flow; orphaned uploads possible → hourly cleanup job; presigning must use the **public** endpoint host, not the internal Docker hostname.
