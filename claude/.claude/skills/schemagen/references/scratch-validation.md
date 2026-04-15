# Scratch DB validation

Load at Phase 5. Every proposed migration round-trips up → down → up against a throwaway Postgres before being returned to the user. A migration that has not survived the round-trip is not a proposal — it is a guess.

## Round-trip script

```bash
set -euo pipefail

CONTAINER=schemagen-scratch
PORT=55432

cleanup() { docker rm -f "$CONTAINER" >/dev/null 2>&1 || true; }
trap cleanup EXIT

docker run --rm -d --name "$CONTAINER" \
    -e POSTGRES_PASSWORD=scratch \
    -p "${PORT}:5432" \
    postgres:16-alpine >/dev/null

until docker exec "$CONTAINER" pg_isready -U postgres >/dev/null 2>&1; do
    sleep 1
done

export PGHOST=localhost PGPORT=$PORT PGUSER=postgres PGPASSWORD=scratch

# 1. Apply existing schema baseline (project-specific — adapt command below)
schema/create_schema.sh

# 2. (Optional) seed representative data to catch constraint violations early
# psql -f scratch/seed.sql

# 3. Apply proposed up migration
psql -v ON_ERROR_STOP=1 -f "$UP_FILE"

# 4. Apply proposed down migration
psql -v ON_ERROR_STOP=1 -f "$DOWN_FILE"

# 5. Re-apply up to verify idempotency after down
psql -v ON_ERROR_STOP=1 -f "$UP_FILE"

echo "Round-trip OK"
```

## Verification checks beyond "psql did not error"

psql exiting 0 only proves the SQL parsed and executed. Also verify:

### Column structure match
After step 5, compare the column definition against intent:
```sql
\d+ {{schema}}.{{table}}
```
Confirm type, nullability, default, constraint names.

### Index existence
```sql
SELECT indexname, indexdef FROM pg_indexes
 WHERE schemaname = '{{schema}}' AND tablename = '{{table}}';
```

### Foreign key existence
```sql
SELECT conname, pg_get_constraintdef(oid)
  FROM pg_constraint
 WHERE conrelid = '{{schema}}.{{table}}'::regclass AND contype = 'f';
```

### Down migration actually reverts
After step 4, the table state should be identical to the pre-up state. Easiest check:
```sql
-- Before up:
\d+ {{schema}}.{{table}} > before.txt
-- After down:
\d+ {{schema}}.{{table}} > after.txt
-- diff before.txt after.txt  -- must be empty
```

If not empty, the down migration is incorrect. Fix and re-run.

## When scratch validation does NOT prove safety

State this explicitly in the summary. Scratch validation confirms:
- The SQL parses.
- The up migration applies against a fresh schema.
- The down migration applies after the up.
- Re-applying up after down succeeds.

It does NOT confirm:
- The migration is safe under production write load (long-lock risk).
- The migration is safe at production table size (seconds vs hours).
- Representative production data does not violate a new constraint.
- Application code survives the change (that's the ripple table's job).

For high-risk migrations (populated table, NOT NULL backfill, type change, large indexes), explicitly recommend:
- `EXPLAIN` on the ALTER to estimate time.
- A staging-DB test with production-shaped data volume.
- `CREATE INDEX CONCURRENTLY` instead of blocking index creation.
- `lock_timeout` and `statement_timeout` settings during the migration.

## If validation fails

Do not propose the failing migration. Report:
- Which step failed (up/down/re-up).
- The psql error verbatim.
- The most likely cause (missing object, constraint violation, syntax error).
- A proposed fix.

Re-run after fixing. Repeat until round-trip is clean.
