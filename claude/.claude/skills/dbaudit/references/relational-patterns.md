# Relational patterns, domain modelling, multi-tenancy

Concerns: whether the entity model respects its domain, whether multi-tenancy is enforced consistently, whether log and entity tables are correctly separated, and whether key strategy is coherent.

## Patterns to flag

### Broken or leaky multi-tenancy
- A table that belongs to a tenant-scoped aggregate but has no `tenant_id` column and no FK to an ancestor that carries one. A join path that terminates at a global table lets rows of one tenant be reachable via the wrong tenant.
- `MODELLING`, severity `high` on entity tables; `critical` if the table holds user-reachable data (PII, site records, events).
- Verify ancestry by following FKs upward to the nearest table that contains `tenant_id`. If no such path exists, the table is effectively global — flag.

### Inconsistent tenant-scoping keys
- Some tables use `UNIQUE (key, tenant_id)` (natural key scoped to tenant), others use `UNIQUE (key)` globally — for the same conceptual "key" attribute. Silent cross-tenant collision or tenant leak.
- `MODELLING`, severity `high`.

### Missing natural key on an entity table
- Entity table with only a surrogate PK and no `UNIQUE` on any business attribute. Duplicate rows cannot be prevented; upserts become guesswork.
- Cross-reference this repo's memory: "every entity table carries both a surrogate PK and a UNIQUE natural key (log tables exempted)".
- `MODELLING`, severity `high` on entity tables; do not flag on log tables.

### Natural key on a log table
- A `*_log` / `*_history` / `*_audit` table carrying a `UNIQUE` natural key. Log tables are append-only and should accept duplicates — uniqueness on content prevents re-emitting an observation.
- `MODELLING`, severity `medium` (often hides a latent modelling bug — the table is really an entity, not a log).

### Derived/denormalised column duplicating another table's truth
- Column in table A storing a value also present in table B (e.g. `site.country_name` when `site.country_id` already references `country.name`). Either the derivation is intentional cached denormalisation (flag as needing documented refresh policy) or it is an accident (flag as drift risk).
- `MODELLING`, severity `medium`.

### Duplicate / parallel concepts
- Two tables modelling the same concept — e.g. `appliance_status` and `connectivity_status` both tracking online/offline for the same appliances.
- Two FK paths to the same parent via different intermediate tables (diamond in the ER graph) without a documented invariant that the two paths must agree.
- `MODELLING`, severity `medium`–`high`.

### Enumerated-lookup inconsistency
- Some enumerated domains are stored as lookup tables (`site_status`, `site_type`, `country`), others as `CHECK` constraints, others as free-form text. Same concept modelled three ways across the schema.
- `COHERENCE` label (not MODELLING) — flag with reference back here for context.
- When one domain changes representation between tables that should use the same vocabulary (e.g. "priority" as lookup in `event_priority` but as `VARCHAR` elsewhere), that is `MODELLING`, severity `medium`.

### Missing audit columns
- Entity table without `created_at` / `created_by` (or the repo's equivalent). Cannot answer "who and when" during incident response.
- `MODELLING`, severity `low` by default, `medium` if the table is part of a user-visible audit surface.
- Project convention check: in this repo most tables carry `created_at TIMESTAMPTZ DEFAULT NOW()` and `created_by VARCHAR(100)`. A table missing them is a COHERENCE finding as well — flag under the primary label only.

### Missing `updated_at` on a mutable entity
- Entity with updates allowed (no `*_log` / append-only shape) but no `updated_at` column. Downstream systems cannot cheaply detect change.
- `MODELLING`, severity `low`; `medium` if change-data-capture is used.

### Soft-delete vs hard-delete inconsistency
- Some entity tables expose `deleted_at` / `is_deleted`, others rely on hard DELETE. Mixed policies make queries hostile: every SELECT must know which strategy applies.
- `MODELLING`, severity `medium`.

### FK into a log table
- An entity table with a FK pointing at a `*_log` / `*_history` table. Log tables are append-only and their PKs are event ids — referencing an event as a first-class relation usually indicates a missing entity (the "state" that the log records) that should be the FK target instead.
- `MODELLING`, severity `medium`.

### Entity boundary violation
- A single table carrying attributes that belong to two distinct lifecycles (e.g. "user" columns and "subscription" columns in one row), or a single row representing what the domain treats as many (a JSON column holding a list that should be a child table).
- `MODELLING`, severity `medium`–`high` depending on how tightly coupled queries become.

### Junction table missing a composite PK
- Many-to-many table (e.g. `tenant_user`, `user_skill`) whose PK is a surrogate `id` but no composite UNIQUE on the two FK columns. Same pair can be inserted twice.
- `INTEGRITY` label primarily; cross-reference `constraints-and-integrity.md`. MODELLING only if the table is structurally wrong (e.g. the "many-to-many" is really many-to-one).

## Safe patterns (not findings)

- Entity table: surrogate `id BIGINT` + `tenant_id BIGINT NOT NULL` + `UNIQUE (key, tenant_id)` + FK to `tenant(id) ON DELETE CASCADE`.
- Log table: surrogate `id` + `tenant_id` + `created_at` + **no** unique natural key.
- Junction table: `UNIQUE(a_id, b_id)` plus a surrogate PK for ergonomic FK inbound references (or a composite PK on `(a_id, b_id)` if no inbound FKs).
- Lookup tables with i18n: `code VARCHAR UNIQUE NOT NULL` + `sort_order` + a companion `i18n_<table>` table for translations.

## Evidence required

- The full column list of the suspect table, enough of its FKs to show (or not show) the tenant path.
- For "parallel concept" findings, the declarations of both tables and their column lists side-by-side.
- For "missing natural key" findings, confirmation that the table is classified as entity (not log) via its name + column shape.

## Severity cues

| Cue | Suggested severity |
|---|---|
| Entity table with no path to `tenant_id` | High–Critical |
| Entity table without a unique natural key | High |
| Log table with a unique natural key | Medium |
| Parallel concept modelled twice | Medium–High |
| Denormalised cached column with no refresh policy | Medium |
| Mixed soft-delete policy across entities | Medium |
| Missing `created_at`/`created_by` on an entity | Low–Medium |
| Missing `updated_at` on a mutable entity | Low |
