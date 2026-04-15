# Phase E — Self-audit checklist

Tick every item `[x]` or `[ ]`. For every `[ ]`, return to Phase B or C, fix, and re-audit. Do not deliver the response with any item unchecked.

```
[ ] Every function in the spec's Public Interface is implemented
[ ] Every step in the spec's Internal Logic has corresponding code with a linking comment
[ ] Every row in the spec's Error Table has: (a) a handling code path, (b) a unit test
[ ] Every Gherkin scenario has a corresponding test function
[ ] Every field in the spec's Data Model has: (a) struct field, (b) validation, (c) test
[ ] All JSON tags match the spec's API contract field names exactly
[ ] All HTTP status codes match the spec's Response section exactly
[ ] All error code strings match the spec's Error Table exactly
[ ] No // TODO, // FIXME, // ..., or placeholder implementations remain
[ ] No panic() outside of main
[ ] Every DB query uses parameterised inputs ($1, $2, ...) — no string concatenation
[ ] Every external call has a timeout derived from the spec's latency targets
[ ] Every exported function has a GoDoc comment (British English)
[ ] context.Context is the first parameter of every I/O function
[ ] All resources cleaned up via defer (rows.Close, tx.Rollback, file.Close, ...)
[ ] File paths match the Phase A6 plan exactly — no extra files, no missing files
[ ] Test file count matches implementation file count (1:1 where logic exists)
[ ] Shared types from Phase 2 are imported, not redefined
[ ] Cross-cutting policies from Phase 4 applied (error shape, auth, logging, config, metrics)
[ ] Phase 1 assumptions respected — no code contradicts a recorded assumption
[ ] Build-context interfaces consumed correctly — no signature mismatches with prior components
[ ] Implementation lives in internal/; nothing is exported from pkg/ unless required
[ ] Only essential interfaces and types are exported
[ ] All comments use British English (behaviour, optimisation, colour, serialise, initialise)
[ ] go build, go test -cover (≥70%), and golangci-lint run all pass (Phase C4 verification)
[ ] Migrations are paired (up.sql + down.sql) and the down is a true reverse
```

## Output format for Phase E

Reproduce the checklist with `[x]`/`[ ]`. For any item initially unchecked, add a one-line note explaining what was fixed (and in which Phase B/C file) before the final delivery.
