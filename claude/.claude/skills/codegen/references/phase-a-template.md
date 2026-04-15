# Phase A — Spec Extraction & Pre-Flight

Produce all seven artefacts in order. Do not proceed to Phase B with unresolved blockers in A5 or A7.

## A1 — Playbook Step Identification

```
| Aspect              | Detail                                        |
|---------------------|-----------------------------------------------|
| Playbook step #     | <number from phase-5-playbook.md>             |
| Component name      | <as named in phase-3 or phase-4>              |
| Spec location       | <phase-3 section X / phase-4 section Y>       |
| Spec type           | <domain component / cross-cutting concern>    |
```

## A2 — Extracted Contract Summary

```
| Aspect            | Detail                                      |
|-------------------|---------------------------------------------|
| Component         | <name>                                      |
| Package path      | <e.g. internal/domain/component>            |
| Public symbols    | <exported symbols + one-line justification> |
| Dependencies      | <internal packages + external libs>         |
| DB tables touched | <reads: X, writes: Y>                       |
| Events produced   | <list or "none">                            |
| Events consumed   | <list or "none">                            |
| Error codes       | <every code from the spec error table>      |
```

## A3 — Cross-Cutting Policies Applied

```
| Phase 4 Spec              | Impact on this component                     |
|---------------------------|----------------------------------------------|
| Global Error Handling     | <error response shape, logging policy>       |
| Auth Middleware           | <JWT validation on endpoints X, Y>           |
| Logging & Observability   | <metrics to emit, trace ID propagation>      |
| Configuration             | <env vars consumed>                          |
```

Include every Phase 4 spec that materially affects this component; omit rows that genuinely do not apply.

## A4 — Shared Types Referenced

```
- SharedType1 (defined in internal/shared/types.go) — used as <field / param / return>
- SharedType2 ...
```

Types must be imported, never redefined (R8, R13).

## A5 — Dependency Interface Check

```
| Dependency         | Expected interface / function     | Present in build context? |
|--------------------|-----------------------------------|---------------------------|
| UserRepository     | GetByID(ctx, id) (*User, error)   | yes / no                  |
```

If any row is "no": state whether you will generate an interface stub (and why the stub is safe) or stop and report a blocking gap.

## A6 — File Plan

List every file to be produced with absolute package path and one-line purpose:

```
internal/domain/component/errors.go       — Typed error definitions and code registry
internal/domain/component/model.go        — Structs and validation methods
internal/domain/component/repository.go   — DB access (queries, transactions)
internal/domain/component/service.go      — Business logic orchestration
internal/domain/component/handler.go      — Echo HTTP handler + Swag annotations
internal/domain/component/wire.go         — Constructors (NewService, NewHandler)
migrations/<YYYYMMDDHHMMSS>_<name>.up.sql
migrations/<YYYYMMDDHHMMSS>_<name>.down.sql
internal/domain/component/service_test.go
internal/domain/component/handler_test.go
internal/domain/component/repository_integration_test.go  // build tag: integration
```

Phase B must produce exactly these files — no more, no fewer.

## A7 — Spec Gaps Detected

For each ambiguity, contradiction, or missing detail:

```
SPEC-GAP-1: <gap description>
Resolution: <what you will do>
Blocking?: yes / no
```

If no gaps: write exactly "None detected." If any gap is blocking, stop and report to the user before Phase B.
