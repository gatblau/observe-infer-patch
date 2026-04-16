# Coherence & conventions

Concerns: cross-file consistency that no single file can get wrong on its own — naming, grouping, deploy order, dead code, documentation. Low per-finding severity but high aggregate cost.

## Patterns to flag

### Naming drift: tables
- Mixed singular and plural: `site`, `tenants`, `events`. Pick one.
- Mixed casing: `site_status` vs `siteStatus` vs `SiteStatus`.
- Prefixes or suffixes inconsistent: `tbl_site`, `site_tbl`, `site`. Pick one.
- `COHERENCE`, severity `info`–`low` individually; `medium` when aggregated (e.g. "12 entity tables plural, 8 singular").

### Naming drift: columns
- `created_at` vs `createdAt` vs `create_time` vs `created_datetime`.
- `tenant_id` vs `tenantid` vs `tenantId`.
- FK columns sometimes `<parent>_id`, sometimes `<parent>id`, sometimes aliased (`owner_id` for a `user` parent — legitimate, but should be documented).
- `COHERENCE`, severity `low` individually.

### Naming drift: constraints
- Mixed FK naming: `site_tenant_id_fk`, `fk_site_tenant`, `tenant_fk`. Breaks error-message grep tooling.
- Mixed index naming: `idx_site_tenant_key`, `site_tenant_key_idx`, `i_site_tenant`.
- `COHERENCE`, severity `low`.

### Naming drift: functions
- Mixed prefixes: `ins_site`, `insert_site`, `upsert_site`, `set_site`. Readers cannot predict function names.
- Action vocab inconsistent: `get_*` vs `find_*` vs `select_*`.
- `COHERENCE`, severity `low`–`medium`. Flag when the project's docs claim a convention and practice diverges.

### Sequence-start policy drift
- Project convention: seed-reserved ids live in `[1, 1000)`, app-generated in `[1000, ∞)` (`CREATE SEQUENCE … START 1000`). One sequence starts at 1 — mixes seed and runtime ids, causing collisions if a seed is added later.
- `COHERENCE`, severity `medium` (collision risk, not just style).

### Idempotency-wrapper drift
- Project convention: every `CREATE TABLE` wrapped in `DO $$ BEGIN IF NOT EXISTS(SELECT relname FROM pg_class WHERE relname = '…') THEN … END $$;` for re-run safety. One table declared without the wrapper.
- `COHERENCE`, severity `medium` — re-running the deploy script fails.

### Deploy-order hazard
- `manifest.json` deploy order declares table B before table A, but A has a FK to B. Or a function is declared before the table it references. Fresh deploys fail on clean DB.
- `COHERENCE` (or `CORRECTNESS` — prefer `CORRECTNESS` when it is a hard deploy failure), severity `high`.
- Evidence: the manifest's `commands[].scripts[]` order vs the actual FK/reference dependencies.

### `deploy-schema` declares a file but the file isn't at the declared path
- Manifest references `tables/misc/locale.sql` but there's no such file (or it moved during refactor). Fresh deploy fails.
- `CORRECTNESS`, severity `high`.

### Entry missing `COMMENT ON`
- `COMMENT ON TABLE …` / `COMMENT ON COLUMN …` absent on entity tables where the project's `documents/` clearly expects documentation. Zero inline context for new engineers and for generated diagrams.
- `COHERENCE`, severity `info`–`low`.

### Orphan SQL file
- `tables/legacy/*.sql` or `funcs/deprecated_*.sql` on disk but not declared in `manifest.json`. Dead code masquerading as live.
- `COHERENCE`, severity `low`.

### Function in `funcs/` not listed in `dropfxs.sql`
- Upgrade drops functions declared in `dropfxs.sql`, then recreates them from `funcs/*.sql`. A function missing from `dropfxs.sql` but present in `funcs/` survives across upgrades as a stale overload.
- `CORRECTNESS`, severity `medium` (upgrade hazard).

### Function in `dropfxs.sql` that doesn't exist in `funcs/`
- `dropfxs.sql` drops `old_fn_name` that is no longer declared anywhere. Harmless but stale.
- `COHERENCE`, severity `info`.

### Duplicate entries in the deploy manifest
- Two `scripts` with different names but the same `file`. Runs the same SQL twice — usually a merge artefact.
- `COHERENCE`, severity `low`.

### Cross-directory naming drift
- `schema/v1/command/tables/<domain>/<table>.sql` expected → actually `schema/v1/command/tables/<table>.sql` (domain subdir missing). Conventions drift as the codebase grows.
- `COHERENCE`, severity `info`–`low`.

### Manifest entry typos
- Manifest `scripts.name` says `"event type"` twice for two different files (`event_priority.sql` and `event_type.sql`) — readable logs conflate them.
- `COHERENCE`, severity `info`.

### Seed/test data colliding with runtime id range
- Seed file inserts `id = 1500` (app-range) while the convention reserves `[1, 1000)` for seeds. First app insert collides on the sequence.
- `COHERENCE`/`CORRECTNESS`; prefer `CORRECTNESS`, severity `high`.

### Missing i18n entry for a newly added enum
- Table `event_category` has values `1..5` but `i18n/events/event_category.sql` only has translations for `1..3`. UI shows raw ids for the rest.
- `COHERENCE`, severity `low`.

### Inconsistent audit column set
- Most tables have `created_at`, `created_by`, `updated_at`. A few have only `created_at`. A couple have `modified_at` (different name for the same concept).
- `COHERENCE`, severity `low`–`medium`. Cross-reference `relational-patterns.md` if the missing column changes a domain invariant.

### `TIMESTAMPTZ DEFAULT NOW()` vs `TIMESTAMP DEFAULT CURRENT_TIMESTAMP` drift
- Half the tables use one, half use the other — functionally equivalent in modern PG, but readers constantly pause to check.
- `COHERENCE`, severity `info`.

## Safe patterns (not findings)

- Every table file wrapped in `DO $$ BEGIN IF NOT EXISTS(...) THEN ... END IF; END $$;` for re-run safety.
- Sequence-start policy documented in one place and applied uniformly.
- Manifest deploy order topologically sorted by dependencies (parents before children, tables before functions, tables before views, views after tables).
- `COMMENT ON TABLE`/`COMMENT ON COLUMN` on every entity surface.
- `dropfxs.sql` includes exactly the functions that `funcs/*.sql` creates.

## Evidence required

- The convention itself — stated in `CLAUDE.md`, `documents/`, or emerging from the first N instances in the repo.
- The outlier — path:line of the divergent instance(s).
- A count if aggregating ("12/20 tables follow the convention, 8 don't — outlier list below").

## Severity cues

| Cue | Suggested severity |
|---|---|
| Deploy-order hazard (referenced object not yet declared) | High |
| Missing file referenced in manifest | High |
| Function in `funcs/` missing from `dropfxs.sql` | Medium |
| Seed id colliding with app-range sequence | High |
| Idempotency wrapper missing on a `CREATE TABLE` | Medium |
| Sequence start inconsistent with project convention | Medium |
| Audit-column set inconsistent across entities | Low–Medium |
| Naming drift (tables, constraints, indexes, functions) | Info–Low individually, Medium aggregated |
| Missing `COMMENT ON` | Info–Low |
| Orphan file on disk, not in manifest | Low |
