# Downstream update ripple analysis

Load at Phase 4. Produces the ripple table in the final output. Every schema change has ripple effects; enumerating them is the main value-add of schemagen over a raw SQL edit.

## Layers to cross-reference

Walk each layer in order. For each layer, decide whether the proposed change touches it and what edit is required.

### 1. Shared types (Phase 2C of the spec)

Location: `docs/spec/phase-2-architecture.md` (section "Shared Types" or equivalent).

A shared type is any Go struct, TS type, or Python dataclass that represents a row of the affected table. Look for:
- Struct field name matching the column name (case-insensitive).
- DB tags referencing the table (`db:"users"`, `pg:"users"`).
- Comment links `// see table users`.

Required edit per change type:
- Additive column → add field.
- Drop column → remove field.
- Rename column → rename field and any JSON/DB tags.
- Type widen → widen field type.
- NOT NULL add → change field from `*T` pointer to `T` value (Go) or remove `Optional[...]` (Python).

### 2. Component Data Model (Phase 3)

Location: `docs/spec/phase-3-components.md` or `docs/spec/components/*.md`. Each component spec has a **Data Model** section containing DDL blocks.

Match the DDL block to the table being changed. Every such block is a spec-level duplicate of the schema — update it to match the post-migration state.

If multiple components reference the same table, all of their Data Model blocks need identical edits.

### 3. Generated code (Phase 3 codegen output)

Location: typically `internal/domain/<component>/repository.go`, `model.go`, or equivalent.

Required action: re-run codegen for each affected component. Do not hand-edit generated files.

Output format for ripple table:
```
/codegen step: "<N>: <Component>" mode: regenerate
```

### 4. Handler / API surface

If the changed column is exposed in a request or response body, list:
- The handler file(s) referencing the column.
- The Phase 3 API contract section to revise (request/response schema, OpenAPI path, etc.).
- Whether the API change is backward-compatible.

Additive and widening changes are usually API-compatible. Drop, rename, and narrowing are breaking.

### 5. Validation / business logic

Look for explicit field checks outside the repository layer:
- Validation functions (`validateUser`, schema validators).
- Business rules referencing the column (`if user.Status == "active"`).

Field removal or rename breaks these; list each one.

### 6. Tests and fixtures

- Integration test SQL files in `*/testdata/*.sql`.
- Seed data scripts.
- Factory/builder helpers in test utilities.

Additive changes with defaults often need no fixture edits. Drop/rename/NOT-NULL always do.

### 7. Observability

- Metrics labels derived from the column.
- Log statements including the column value.
- Dashboards / alerts referencing the column name in queries.

Often forgotten; surface explicitly so the user can decide scope.

## Search heuristics

For each column name, run:
- Grep on the exact column name, case-sensitive, across the repo.
- Grep on the CamelCase equivalent (`user_id` → `UserID`).
- Grep on any short alias used in queries (`u.user_id`).

For each table name, additionally grep on:
- The snake_case name (`user_sessions`).
- Any struct named after it (`UserSession`, `UserSessions`).

Do not rely on conventions to skip grepping. Generated code or manual SQL can reference names in unexpected forms.

## Ripple table format

```
| Artefact                          | Location                                         | Required change                  |
|-----------------------------------|--------------------------------------------------|----------------------------------|
| Shared type `User`                | docs/spec/phase-2-architecture.md §Shared Types  | Add `Email string` field         |
| Component spec `UserRepository`   | docs/spec/components/user-repository.md §Data Model | Update DDL block              |
| Generated repository              | internal/domain/user/repository.go               | Regenerate via /codegen          |
| Handler `CreateUser`              | route/user.go                                    | Accept `email` in request body   |
| Integration fixtures              | internal/domain/user/testdata/seed.sql           | Add email values to seed rows    |
| OpenAPI spec                      | docs/openapi.yaml §/users POST                   | Add email property               |
```

Empty layers (no impact) are omitted from the final ripple table but stated explicitly in the summary: "Ripple: no impact on handlers, no impact on observability."
