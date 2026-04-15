# Phase B — Code Generation guide

Produce every file listed in A6 in the sub-order below. Skip sub-phases that do not apply (a middleware component may only need B1, B4, B5, B6).

## Sub-phase ordering

```
B1. errors.go      — Typed error definitions + error code constants
B2. model.go       — Data models and validation
                     (import Phase 2 shared types; do NOT redefine)
B3. repository.go  — Data access (queries, transactions)
B4. service.go     — Business logic orchestration
B5. handler.go     — HTTP handler / NATS consumer / CLI entrypoint
B6. wire.go        — Constructors: NewService, NewHandler, etc.
B7. migrations/    — YYYYMMDDHHMMSS_<name>.up.sql + .down.sql (paired)
```

## Per-file requirements

Every generated file must:

1. Start with `// File: <full/path/from/module/root.go>` as the first line (inside the code block).
2. Declare its package, then imports.
3. Use spec-linking comments at key sites:
   - `// Implements SPEC behaviour step N` above logic that fulfils a numbered step in the spec's Internal Logic
   - `// Error table row: <Condition>` at every error-handling site
   - `// Phase 4: <Spec Name>` where a cross-cutting policy is applied (auth check, error envelope, log correlation ID, config read, metric emission)
4. Give every exported type and function a GoDoc comment in British English.
5. Add Swag annotations (`@Summary`, `@Param`, `@Success`, `@Failure`, `@Router`) on every Echo HTTP handler, matching the spec's API contract exactly.

## TechOps requirements for generated code

Apply these patterns in every applicable file:

- **Resilience (R9):** explicit timeouts via `context.WithTimeout`; retry with exponential backoff + jitter for transient errors; circuit breaker state where the spec calls for it; backpressure on overwhelmed downstreams.
- **Security (R10):** input validation at the boundary; parameterised SQL (`$1`, `$2`, ...); auth checks at the handler layer; never log secrets or PII; constant-time comparisons for secret material.
- **Production (R4, R11):** structured JSON logs with correlation/trace IDs; `context.Context` as first parameter on every I/O function; no hardcoded config; typed errors.
- **Safe change (R12):** backward-compatible request/response shapes where the spec allows; error messages that do not leak internal state; small functions.

## Migrations

- Paired files: `<timestamp>_<name>.up.sql` and `<timestamp>_<name>.down.sql`. Timestamp in `YYYYMMDDHHMMSS`.
- `down.sql` must be a true reverse of `up.sql`. For destructive changes (DROP, data-losing ALTER), include a rollback note comment at the top of `up.sql`.
- Reference the exact table/column names from the spec's Data Model section; do not infer names from patterns.

## What NOT to produce in Phase B

- Documentation files (`doc.go`) — that is Phase D.
- Test files — that is Phase C.
- README or design notes — not produced by this skill.
- Any file not listed in A6. If Phase B discovers a need for an unplanned file, return to Phase A and revise A6 before generating it.
