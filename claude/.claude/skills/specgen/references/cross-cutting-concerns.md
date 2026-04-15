# Phase 4 — cross-cutting concerns

Produce one component-style spec (using `component-spec-template.md`) for each concern below. These specs are treated as first-class components and appear in the Phase 2B inventory.

| Concern | Spec must cover |
|---|---|
| Authentication & Authorisation | Token format, validation flow, RBAC/ABAC model, permission checks, key rotation |
| Error Handling | Standard error envelope, error code registry, panic recovery, error-to-HTTP mapping |
| Logging | Format (JSON), required fields, correlation ID propagation, log levels, redaction rules |
| Metrics | Prometheus metric names, label conventions, histogram buckets, cardinality limits |
| Tracing | OpenTelemetry span naming, context propagation, sampling rate, exporter configuration |
| Configuration | Loading order (env > file > default), validation on startup, required vs optional, secret handling |
| Database Migrations | Tool choice (e.g. golang-migrate), naming convention, CI vs startup execution, rollback strategy |
| Health Checks | `/healthz` (liveness) and `/readyz` (readiness) definitions, check dependencies, timeouts |
| Rate Limiting | Algorithm (token bucket / sliding window), per-endpoint limits, headers returned, bypass rules |
| Pagination | Cursor vs offset strategy, request/response schema, max page size, stable ordering |
| CORS | Allowed origins, methods, headers, max-age, credential handling |
| Input Validation | Library, validation tags, sanitisation rules, max body size, error shape |
| Graceful Shutdown | Signal handling, drain timeout, connection cleanup order, in-flight request handling |

If any concern is explicitly out of scope for the project, record that decision in the Phase 1A Assumptions Register rather than silently omitting it.
