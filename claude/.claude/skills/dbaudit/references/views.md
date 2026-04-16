# Views

Concerns: whether a view's shape, join set, and filter predicates match its stated purpose; whether it is a safe API surface; whether it should be materialised.

## Patterns to flag

### View referencing columns that don't exist on the base table
- `CREATE VIEW ... AS SELECT t.renamed_col FROM t;` where `t.renamed_col` is not in `t`. Works until the view is refreshed/recreated; may run for months as a ghost.
- `CORRECTNESS`, severity `high`. Confidence `regex-confidence` unless the base table is inventoried in the same pass.

### View used as a public API without a contract
- View referenced by PL/pgSQL functions, application code, or another view — but its column list has changed over time with no migration discipline. Callers break on rename/remove.
- `MODELLING` (API boundary), severity `medium`.

### View joining tables without indexes on the join keys
- View joins `a ⋈ b ON a.b_id = b.id` where `a.b_id` has no index. Every SELECT from the view does a hash/merge join of `a`.
- `PERFORMANCE`, severity `medium`. Cross-reference `indexing-and-performance.md`.

### View applying a filter the base table doesn't index
- `CREATE VIEW active_x AS SELECT ... WHERE deleted_at IS NULL` where `deleted_at` is unindexed and the base table has many rows. Seq scan on every view access.
- `PERFORMANCE`, severity `medium`. Partial index is usually the fix.

### View performing aggregation recalculated every read
- View aggregates over millions of rows (`SUM`, `COUNT`, `AVG`) to produce a dashboard value. Every caller recomputes it.
- `PERFORMANCE`, severity `medium`–`high` at scale. Consider materialised view + refresh strategy.

### `SELECT *` in a view definition
- `CREATE VIEW v AS SELECT * FROM t;` — the view's column list is frozen at creation. If `t` grows columns, the view doesn't expose them (silent under-exposure); if `t` renames columns, the view breaks the next time it's recreated.
- `COHERENCE`, severity `low`–`medium`. Prefer an explicit column list.

### View filtering by tenant but not enforcing it at callers
- View with a `WHERE tenant_id = current_setting('app.tenant_id')::bigint`-style predicate used by tenant-scoped queries. Works, but easy to miss the `SET app.tenant_id` on a caller — silent cross-tenant leak.
- `SECURITY`, severity `high` when tenant-scoping is a correctness invariant; `medium` when it's defence-in-depth.
- Prefer Postgres Row-Level Security (RLS) with a policy on `current_setting` — enforced by the engine, not by the view.

### View with correlated subquery per row
- View's projection contains a `(SELECT … FROM other WHERE other.x = v.x)` subquery. Executed per row. For 1000-row outputs, becomes 1000 extra queries.
- `PERFORMANCE`, severity `medium`. LATERAL join or aggregation usually solves it.

### View hiding a multi-tenancy boundary error
- View joining two tables where one is tenant-scoped and the other is global, without including `tenant_id` in the final projection. Callers lose the ability to filter by tenant downstream.
- `MODELLING`+`SECURITY`; prefer `MODELLING`, severity `medium`.

### Materialised view without a refresh story
- `CREATE MATERIALIZED VIEW mv AS ...` with no trigger, scheduled job, or manual doc explaining when `REFRESH MATERIALIZED VIEW mv` runs. Silently stale.
- `CORRECTNESS`, severity `medium`–`high`.

### Materialised view with `REFRESH` blocking writes
- `REFRESH MATERIALIZED VIEW mv` without `CONCURRENTLY` takes an ACCESS EXCLUSIVE lock. For dashboards, outage during refresh.
- `PERFORMANCE`, severity `medium`. Fix requires a UNIQUE index on the MV and `REFRESH … CONCURRENTLY`.

### Orphan view
- View present in the repo but not referenced by any function or (grep-checked) application code, and not documented as an external API.
- `COHERENCE`, severity `low`. Candidate for deletion.

### View names colliding with base tables semantically
- `user_permissions` view AND `user_permission` table co-exist. Every reader must disambiguate.
- `COHERENCE`, severity `low`.

## Safe patterns (not findings)

- Explicit column list; view acts as a stable API surface, evolved via migrations.
- Join keys indexed on both sides (usually PK + FK index).
- Materialised view with: UNIQUE index on the natural key, trigger or scheduled `REFRESH … CONCURRENTLY`, monitoring of staleness.
- RLS policy used for tenant scoping instead of (or in addition to) a view predicate.

## Evidence required

- The view definition in full.
- Every referenced base table's column list (enough to verify columns exist).
- Every `CREATE INDEX` on the referenced tables (for join/filter findings).
- Every caller (function / view / grep-found app call) referencing the view.

## Severity cues

| Cue | Suggested severity |
|---|---|
| View references a column that doesn't exist | High |
| Materialised view with no refresh story | Medium–High |
| Tenant filter applied only in view, not RLS | Medium–High |
| Join on unindexed keys in a hot view | Medium |
| Aggregation per read on a large table | Medium–High |
| `SELECT *` in view definition | Low–Medium |
| Orphan view | Low |
