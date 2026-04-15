# Schema change classification

Use this reference at Phase 1 to classify the requested change. Classification drives template choice (Phase 3) and ripple-depth (Phase 4).

## Additive (low risk)

Safe, reversible, no data at risk.

- `ADD COLUMN` nullable, no default OR with constant default (PG11+ fast path).
- `CREATE TABLE`.
- `CREATE INDEX` (prefer `CONCURRENTLY` on populated tables; note this cannot run inside a transaction).
- `CREATE EXTENSION IF NOT EXISTS`.
- Adding a new enum value with `ALTER TYPE ... ADD VALUE`.

Down-migration: straightforward `DROP`.

## Extension (low risk with caveats)

- Widening a type (`VARCHAR(100)` → `VARCHAR(200)`, `INT` → `BIGINT`): generally safe, but rewrites the table on some engines — note if the table is large.
- Raising a precision (`NUMERIC(10,2)` → `NUMERIC(14,2)`).

Down-migration: may truncate data; always mark as lossy.

## Destructive (high risk)

Requires ROLLBACK CONSIDERATIONS block in up.sql and explicit user approval.

- `DROP COLUMN` / `DROP TABLE` / `DROP INDEX`.
- `ALTER COLUMN TYPE` narrowing (`BIGINT` → `INT`, `TEXT` → `VARCHAR(50)`).
- Adding `NOT NULL` to a populated nullable column.
- Removing an enum value (Postgres: only possible by recreating the type).
- Changing a column default when application relies on existing default.

Down-migration restores structure but cannot restore data. Mark explicitly.

## Rename (non-destructive, ripple-heavy)

- `ALTER TABLE ... RENAME COLUMN` / `RENAME TO`.
- `ALTER INDEX ... RENAME TO`.

Breaks every consumer referencing the old name. Ripple table must be exhaustive — every spec, every `SELECT`, every ORM binding.

## Constraint

- Add/drop `FOREIGN KEY`, `UNIQUE`, `CHECK`, `PRIMARY KEY`.
- Adding FK or UNIQUE on a populated table may fail if existing rows violate — always validate on scratch DB with representative data if possible.
- Dropping a constraint is reversible only if the constraint was explicitly named (otherwise the generated name must be looked up).

## Multi-step patterns

Certain single-intent changes must be split into multiple migrations:

### NOT NULL on populated column
1. Add column nullable with a default (or leave nullable).
2. Backfill existing rows.
3. Add NOT NULL constraint.

### Column rename with zero-downtime consumers
1. Add new column.
2. Dual-write in app.
3. Backfill.
4. Switch reads.
5. Drop old column.

Schemagen proposes the migrations; the user decides whether to collapse based on deployment constraints.

### Type narrowing with validation
1. Add CHECK constraint enforcing the narrower range.
2. Validate (may be slow).
3. Alter type.
4. Drop CHECK.

## Classification output

State in the final summary:
```
Classification: <Additive|Extension|Destructive|Rename|Constraint>
Multi-step: <yes|no>  (if yes, list the steps)
Reversible: <fully|structure-only|not>
```
