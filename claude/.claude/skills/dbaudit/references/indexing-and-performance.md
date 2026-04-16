# Indexing & performance

Concerns: indexes that are missing, duplicated, redundant, or mis-shaped; view and function query shapes that provoke seq scans; `ON DELETE CASCADE` chains without supporting indexes.

## Patterns to flag

### Missing index on a FK column
- Postgres does **not** auto-index the child side of a FK. Every `UPDATE`/`DELETE` on the parent has to tablescan the child to enforce referential integrity (and to cascade). At scale this is the single most common perf bug in a PG schema.
- `PERFORMANCE`, severity `high` on any FK whose parent is mutable at scale (e.g. `tenant`, `site`); `medium` on slowly-changing lookups (`country`, `site_status`).
- Exception: the FK column is also the leading column of an existing index — then it's covered. Always check index list before flagging.

### `ON DELETE CASCADE` without an index on the FK
- Upgrade of the above — deleting a parent row triggers the tablescan plus the cascade. On large child tables this blocks the DELETE for minutes.
- `PERFORMANCE`, severity `high`.

### Duplicate index — same columns, same order
- `CREATE INDEX a_x_idx ON a(x); CREATE INDEX a_x_idx2 ON a(x);` — one of them is pure bloat (disk + write amplification).
- `PERFORMANCE`, severity `medium` (write amplification on a hot table) or `low` (lookup table).
- **Note on column order:** indexes on `(a, b)` and `(b, a)` are NOT duplicates — they serve different query shapes. Only flag as duplicate when the column list and order match.

### Redundant index — subset of a composite
- `CREATE INDEX i1 ON t(a); CREATE INDEX i2 ON t(a, b);` — `i2` serves every query `i1` serves (leading-column rule). `i1` is redundant unless the planner specifically needs a smaller index (rare).
- `PERFORMANCE`, severity `low`–`medium`.

### Two indexes with the same columns in different orders
- `CREATE INDEX i1 ON t(tenant_id, key); CREATE INDEX i2 ON t(key, tenant_id);`. Both cost write amplification. One of them is probably unused.
- These are not redundant automatically — decide by query shape:
  - Queries filtering by `tenant_id` alone or `tenant_id + key` → `i1` is needed.
  - Queries filtering by `key` alone (rare on a tenant-scoped entity) → `i2` would be used.
  - Queries like `WHERE key = $1 AND tenant_id = $2` can use either.
- Flag this as "two orders present; likely one is unused" and recommend the user confirm via `pg_stat_user_indexes` before dropping.
- `PERFORMANCE`, severity `low`–`medium`.

### Index on a boolean alone
- `CREATE INDEX t_is_active_idx ON t(is_active);` — at most two distinct values, planner usually prefers seq scan. If the hot predicate is `WHERE is_active = true AND other_col = ?`, a partial index on `other_col WHERE is_active` is what's actually useful.
- `PERFORMANCE`, severity `low`.

### Partial-index opportunity
- Hot filter `WHERE deleted_at IS NULL` applied on most queries against a table with many deleted rows. A partial index `WHERE deleted_at IS NULL` is smaller, tighter, and faster.
- `PERFORMANCE`, severity `low`–`medium`.

### Covering-index opportunity / wrong column order
- Composite index `(a, b)` when the hot query filters `WHERE b = ?` alone. The index can't be used with leading-column rule violated.
- `PERFORMANCE`, severity `medium`.

### Index on a low-cardinality leading column
- `CREATE INDEX t_status_id_idx ON t(status_id);` where `status_id` has 5 distinct values on a table with 10M rows. Postgres ignores the index for anything other than highly selective value lookups.
- `PERFORMANCE`, severity `low`. Consider composite `(status_id, created_at)` or partial indexes for the hot statuses.

### Index on an expression rather than the column
- `CREATE INDEX t_lower_email ON t(LOWER(email));` is correct **if and only if** all queries use `WHERE LOWER(email) = ?`. If the app sometimes queries `WHERE email = ?`, that query skips the index. Flag when inconsistent.
- `PERFORMANCE`, severity `medium`.

### Missing index to support a UNIQUE across large tables
- `UNIQUE (a, b)` implicitly creates a btree on `(a, b)` — not a finding.
- BUT `UNIQUE (a, b)` does not help `WHERE b = ?`. If that query exists, add an index on `b` (or change the composite order).
- `PERFORMANCE`, severity `medium`.

### Indexes on log/append-only tables that only receive writes
- Log table with multiple indexes but read queries only use `(entity_id, created_at DESC)`. Each extra index is pure write cost.
- `PERFORMANCE`, severity `low`.

### Missing `CREATE INDEX … CONCURRENTLY` in migration-safe file
- Migration file that does `CREATE INDEX … ON large_table(…)` (non-concurrently) will take an ACCESS EXCLUSIVE lock. For new-table creation this is fine; for adding an index to an existing hot table it's an outage.
- `PERFORMANCE`, severity `high` when on an existing table; `info` when part of initial table creation.

### Indexed column that's frequently updated
- Every update to `status_id` rewrites all indexes containing it. Pick indexes carefully on hot-update columns.
- `PERFORMANCE`, severity `low`. Often unavoidable; flag for awareness.

### View with unindexed join predicate
- View joins `a` to `b` on `a.b_id = b.id` where `a.b_id` has no index. Every SELECT from the view does hash/merge join over full `a`.
- `PERFORMANCE`, severity `medium`.

### Function body with an unindexable predicate
- PL/pgSQL function body with `WHERE key ILIKE '%' || param || '%'`. Cannot use a btree. For prefix, use `key LIKE param || '%'` + index on `key` (or a `text_pattern_ops` index).
- `PERFORMANCE`, severity `medium`.

### Unbounded query in a function
- Function returning all rows for a tenant with no LIMIT. Caller usually wants pagination, but the function doesn't express it — worst case: 100k rows returned per call.
- `PERFORMANCE`, severity `medium`.

## Safe patterns (not findings)

- `CREATE INDEX idx_site_tenant_key ON site (tenant_id, key);` — composite covering tenant-scoped lookup.
- `CREATE INDEX ... CONCURRENTLY` for adding indexes on live tables (migrations only, not inside a transaction).
- Partial index for hot filter: `CREATE INDEX active_events_idx ON event(created_at) WHERE status = 'open';`.
- A FK column that is also the leading column of an existing composite index — already covered.

## Evidence required

- The full `CREATE TABLE` + every `CREATE INDEX` declared on the table (in the same or a related file).
- For duplicate/redundant findings, both index declarations with file:line.
- For view/function shape findings, the query body plus the declared indexes of every referenced table.

## Severity cues

| Cue | Suggested severity |
|---|---|
| Unindexed FK on a mutable parent | High |
| `ON DELETE CASCADE` with no index on child FK | High |
| Non-CONCURRENT `CREATE INDEX` on a live table in migration | High |
| Exact-duplicate index on hot table | Medium |
| Redundant subset index on hot table | Low–Medium |
| Two orderings of the same columns — one likely unused | Low–Medium |
| Partial-index opportunity on hot predicate | Low–Medium |
| Index on low-cardinality leading column | Low |
| Index on a boolean alone | Low |
| Unbounded SELECT in function body | Medium |
