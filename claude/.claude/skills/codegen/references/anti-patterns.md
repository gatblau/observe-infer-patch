# Anti-patterns — forbidden in generated code

Never produce any of the following. Use as a final filter before Phase E.

| Anti-pattern | Why it is forbidden |
|---|---|
| `// ... remaining implementation similar` | R2 — write every line |
| `// TODO: add validation` / `// FIXME` | R2 — add the validation now or flag `SPEC-GAP` |
| `panic(err)` in library code | R5 — return the error; panics only in `main` bootstrap |
| `interface{}` or `any` when the spec names a concrete type | R6 — use the concrete type |
| `context.Background()` in non-`main` code | R4 — propagate the caller's context |
| `_ = db.Close()` (ignored Close errors) | R5 — at minimum log the error |
| Raw SQL via string concatenation or `fmt.Sprintf` | R10 — parameterise (`$1`, `$2`, ...) |
| Inventing endpoints, fields, events, or subjects not in the spec | R1 — spec is law |
| Skipping tests for "simple" functions | R3 — simple functions get simple tests |
| Adding dependencies not listed in PROJECT CONSTANTS | Flag as `SPEC-GAP` if truly needed |
| Redefining a Phase 2 shared type | R8 — import it |
| Ignoring a Phase 4 cross-cutting spec | R8 — apply it |
| Contradicting a Phase 1 assumption | R8 — respect it |
| Exporting symbols that are not part of the required service interface | R13 — default to unexported |
| Placing implementation under `pkg/` when it is not designed for external import | R13 — use `internal/` |
| American spelling in comments ("behavior", "color", "initialize") | R14 — British English only |
| Swallowing errors with `if err != nil { return nil }` | R5 — wrap and return typed errors |
| Hardcoded URLs, ports, paths, or credentials | R4 — read from Phase 4 Configuration spec |
| Migration `up.sql` without a corresponding `down.sql` | R12 — every change must be reversible |
