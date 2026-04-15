# Phase 6 — self-audit checklist

Tick each item explicitly. If any item fails, fix the relevant spec and re-audit. Do not deliver output with unchecked items.

```
[ ] Every entity has a complete data model with types and constraints.
[ ] Every action has defined inputs, outputs, numbered steps, and errors.
[ ] No banned phrases remain (see banned-phrases.md).
[ ] Every component has ≥3 Gherkin acceptance criteria (happy, edge, error).
[ ] Every component has an error table with ≥2 rows.
[ ] Every cross-component interaction is documented on BOTH sides.
[ ] Build order (Phase 2B) is a valid DAG — no circular dependencies.
[ ] Every config value / env var listed with type, default, required flag, and owner component.
[ ] Every spec is self-contained — implementable from its section alone.
[ ] Assumptions register is complete; open questions are truly blocking only.
[ ] Example I/O provided for every component with non-trivial logic.
[ ] Shared types defined once in Phase 2C, referenced by name elsewhere but duplicated into "Shared Context" of each consuming spec.
[ ] Security addressed for every entry point (HTTP handler, NATS subscriber, CLI command).
[ ] Performance targets stated for every latency-sensitive component.
[ ] All specifications use British English spelling and terminology.
[ ] Destructive schema changes (DROP, data-losing ALTER, NOT NULL backfills) surface rollback considerations.
```

Output format for Phase 6: reproduce the checklist with `[x]` or `[ ]` and, for any unchecked item, a one-line note explaining what was fixed or why it remains open.
