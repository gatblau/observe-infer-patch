# Migration templates

Templates per change type. Substitute `{{placeholders}}` and remove anything not needed. Every template has an up/down pair.

## Header convention

Every migration file begins with:
```sql
-- Migration: {{name}}
-- Timestamp: {{YYYYMMDDHHMMSS}} UTC
-- Type: {{Additive|Extension|Destructive|Rename|Constraint}}
```

Destructive / rename migrations additionally include:
```sql
-- ROLLBACK CONSIDERATIONS:
-- {{paragraph describing data loss, consumer breakage, manual recovery steps}}
```

## Additive — add nullable column

**up.sql**
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}}
    ADD COLUMN {{column}} {{type}} {{default_clause?}};
COMMIT;
```

**down.sql**
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}} DROP COLUMN {{column}};
COMMIT;
```

## Additive — create table

**up.sql**
```sql
BEGIN;
CREATE TABLE {{schema}}.{{table}} (
    id           BIGSERIAL PRIMARY KEY,
    {{column}}   {{type}} NOT NULL,
    created_at   TIMESTAMPTZ NOT NULL DEFAULT now(),
    updated_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);
CREATE INDEX idx_{{table}}_{{column}} ON {{schema}}.{{table}}({{column}});
COMMIT;
```

**down.sql**
```sql
BEGIN;
DROP TABLE {{schema}}.{{table}};
COMMIT;
```

## Additive — create index concurrently

Cannot run inside a transaction. No `BEGIN`/`COMMIT`.

**up.sql**
```sql
CREATE INDEX CONCURRENTLY idx_{{table}}_{{column}}
    ON {{schema}}.{{table}}({{column}});
```

**down.sql**
```sql
DROP INDEX CONCURRENTLY IF EXISTS idx_{{table}}_{{column}};
```

## Destructive — drop column

**up.sql**
```sql
-- ROLLBACK CONSIDERATIONS:
-- Dropping column {{table}}.{{column}}. Existing data in this column is lost
-- and cannot be recovered by the down migration. If production data in this
-- column is referenced by consumers or downstream systems, coordinate before
-- applying. The down migration restores the column as nullable; a separate
-- backfill is required to restore values.

BEGIN;
ALTER TABLE {{schema}}.{{table}} DROP COLUMN {{column}};
COMMIT;
```

**down.sql**
```sql
-- NOTE: Data lost on original DROP cannot be recovered by this down migration.
BEGIN;
ALTER TABLE {{schema}}.{{table}} ADD COLUMN {{column}} {{original_type}};
COMMIT;
```

## Destructive — NOT NULL on populated column (three-step)

**Step 1 up:** add column nullable.
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}} ADD COLUMN {{column}} {{type}};
COMMIT;
```

**Step 2 up:** backfill.
```sql
BEGIN;
UPDATE {{schema}}.{{table}} SET {{column}} = {{backfill_expr}} WHERE {{column}} IS NULL;
COMMIT;
```

**Step 3 up:** constrain.
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}} ALTER COLUMN {{column}} SET NOT NULL;
COMMIT;
```

Each step has a matching down migration reversing only its own effect.

## Rename column

**up.sql**
```sql
-- ROLLBACK CONSIDERATIONS:
-- Renaming column breaks every consumer referencing the old name. See ripple
-- table for the list of spec, code, and test artefacts that must be updated in
-- lockstep with this migration.

BEGIN;
ALTER TABLE {{schema}}.{{table}} RENAME COLUMN {{old_name}} TO {{new_name}};
COMMIT;
```

**down.sql**
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}} RENAME COLUMN {{new_name}} TO {{old_name}};
COMMIT;
```

## Constraint — add foreign key

**up.sql**
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}}
    ADD CONSTRAINT fk_{{table}}_{{column}}
    FOREIGN KEY ({{column}}) REFERENCES {{ref_schema}}.{{ref_table}}({{ref_column}})
    ON DELETE {{RESTRICT|CASCADE|SET NULL}};
COMMIT;
```

**down.sql**
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}} DROP CONSTRAINT fk_{{table}}_{{column}};
COMMIT;
```

## Constraint — add unique

**up.sql**
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}}
    ADD CONSTRAINT uq_{{table}}_{{column}} UNIQUE ({{column}});
COMMIT;
```

**down.sql**
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}} DROP CONSTRAINT uq_{{table}}_{{column}};
COMMIT;
```

## Extension — widen type

**up.sql**
```sql
BEGIN;
ALTER TABLE {{schema}}.{{table}} ALTER COLUMN {{column}} TYPE {{new_type}};
COMMIT;
```

**down.sql**
```sql
-- NOTE: Narrowing back to {{old_type}} may truncate values that exceed the old range.
BEGIN;
ALTER TABLE {{schema}}.{{table}} ALTER COLUMN {{column}} TYPE {{old_type}};
COMMIT;
```

## Enum — add value

Cannot be run in a transaction in older PG; PG12+ allows it.

**up.sql**
```sql
ALTER TYPE {{schema}}.{{enum_type}} ADD VALUE IF NOT EXISTS '{{new_value}}';
```

**down.sql**
```sql
-- NOTE: Postgres does not support removing an enum value directly. To fully
-- reverse, recreate the type without the value and update all referencing
-- columns. This down migration is intentionally a no-op; manual recovery is
-- required if reversion is needed.
SELECT 1;
```
