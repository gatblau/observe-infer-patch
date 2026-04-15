---
name: schemagen
description: Produce a database schema change (migration up/down pair) plus the downstream updates it requires — Phase 2C shared types, Phase 3 component data models, and the list of codegen re-invocations needed. Validates the migration against a scratch database before proposing. Trigger on phrases like "add column X to table Y", "new migration for Z", "schemagen", "change schema to support W", or when the user asks for a schema evolution. Do NOT trigger on drift-detection requests (that is syncheck) or on initial schema creation from specs (that is codegen B7).
---

# schemagen — Schema change with downstream updates

Produce the minimal schema change that implements a requested evolution, plus the spec and code updates that ripple from it. Heavy integration with `.claude/rules/layer/sql-safety.md` — this skill is the reason that layer exists.

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Arguments

- `change` (required): description of the desired schema change. Accepts natural language ("add column `email` to `users`"), a target state ("users should have an email column, NOT NULL, unique"), or a Phase 2D env var / Phase 3 Data Model delta from a spec.
- `migrations-dir`: default `schema/` (per this repo's layout; override if the project uses `migrations/`).
- `mode`: `propose` (default — produce migration + spec deltas, do not write) | `write` (write the files after user confirmation on each destructive change).

## Active mode

**Patch + sql-safety.** schemagen writes migration files (`mode: write`) and proposes downstream edits (ripple table). Phase 2 verifies the current schema against `schema/v1/**` before any object is named in the proposed migration; Phase 5 round-trips the migration on a scratch DB before returning. sql-safety is non-optional — destructive changes carry an explicit ROLLBACK CONSIDERATIONS block.

## Hard rules (layered over sql-safety)

- **Every change ships as a pair.** `<timestamp>_<name>.up.sql` + `<timestamp>_<name>.down.sql`. The `down` must be a true reverse of the `up`.
- **Never invent object names.** Tables, columns, indexes, constraints must be verified against existing SQL files before referencing them. If the requested change touches an object not found in `schema/v1/**`, stop and tell the user.
- **Destructive changes surface rollback considerations.** DROP COLUMN, DROP TABLE, ALTER TYPE with data loss, NOT NULL backfills on populated tables — each requires an explicit rollback note at the top of `up.sql` and a warning in the summary.
- **Additive first, then backfill, then constrain.** For NOT NULL additions on populated tables, propose three migrations, not one: (1) add column nullable, (2) backfill, (3) add NOT NULL. The user picks whether to collapse or keep separate based on table size.

## Process

### Phase 1 — Parse the change
Load `references/change-types.md`. Classify the request into one of:

| Type | Examples |
|---|---|
| Additive | Add column (nullable), add table, add index, add extension |
| Extension | Add enum value (Postgres), widen a VARCHAR |
| Destructive | DROP column/table/index, NOT NULL on populated column, ALTER TYPE narrowing |
| Rename | Rename column/table (non-destructive but ripple-heavy) |
| Constraint | Add/drop FK, UNIQUE, CHECK |

State the classification in the output. Destructive and rename changes get extra scrutiny.

### Phase 2 — Verify current state
Read the existing schema for the affected tables. This project organises schema under `schema/v1/{extension,tables,funcs,views,content,i18n}` — read the specific files involved, not the whole tree. Record:

- Exact current column definitions (name, type, nullability, default, constraints).
- Indexes touching the column.
- Foreign keys referencing the column or table.
- Views, functions, or triggers referencing the column.
- Rows count if known (ask the user for production scale if not stated — affects backfill strategy).

If anything the change refers to is not found, stop and list the missing objects. Never infer names from pattern ("there's probably a column named X").

### Phase 3 — Design the migration
Load `references/migration-templates.md`. Construct:

- **Timestamp:** `YYYYMMDDHHMMSS` in UTC at the moment of generation.
- **Name:** short kebab-case description of the change (e.g. `add-email-to-users`).
- **Up SQL:** parameterised, explicit. Include `BEGIN; ... COMMIT;` wrapping where Postgres supports DDL in transactions (it does, except for `CREATE INDEX CONCURRENTLY`).
- **Down SQL:** the true inverse. Where true inverse is impossible (e.g. you cannot recover dropped data), the `down.sql` restores structure but cannot restore data — mark this explicitly at the top of `down.sql` with `-- NOTE: Data lost on original DROP cannot be recovered by this down migration.`

For destructive or rename changes, write the `up.sql` with:
```sql
-- ROLLBACK CONSIDERATIONS:
-- <one paragraph explaining what data is lost, what consumers break,
--  and what manual steps are required if reverting after some time in production>
```

### Phase 4 — Downstream ripple analysis
Load `references/downstream-updates.md`. Cross-reference the change against:

- **Phase 2C Shared Types** in `docs/spec/phase-2-architecture.md` — if the affected table has a corresponding shared type, list the field additions/removals/renames needed.
- **Phase 3 Data Model sections** in `docs/spec/phase-3-components.md` (or `docs/spec/components/*.md`) — list every component spec whose Data Model references the changed table. Those specs need an edit.
- **Codegen components** — list the components whose generated code (`repository.go`, `model.go`) needs regeneration. Give exact `/codegen step: "<N>: <Component>"` invocations.
- **Handler/API impact** — if the change affects request/response JSON shape (e.g. new column exposed via an endpoint), list the handlers that need updating and the Phase 3 API contract section to revise.
- **Test impact** — existing integration tests that exercise the table may need fixture updates.

Produce a ripple table in the output:

```
| Artefact | Location | Required change |
|---|---|---|
| Shared type `User` | docs/spec/phase-2-architecture.md §2C | Add Email field |
| Component spec `UserRepository` | docs/spec/phase-3-components.md §UserRepository | Update Data Model DDL block |
| Code (regenerate) | internal/domain/user/repository.go | /codegen step: "4: UserRepository" |
| Integration test fixtures | internal/domain/user/testdata/*.sql | Add email to seed rows |
```

### Phase 5 — Scratch-DB validation
Load `references/scratch-validation.md`. Run the migration against a throwaway Postgres instance:

```bash
# Start a throwaway Postgres
docker run --rm -d --name schemagen-scratch \
  -e POSTGRES_PASSWORD=scratch -p 55432:5432 postgres:16-alpine

# Wait for ready
until docker exec schemagen-scratch pg_isready -U postgres; do sleep 1; done

# Apply existing schema
# (use the project's provisioner — for this repo, the insight-db-init image or schema/create_schema.sh)
PGHOST=localhost PGPORT=55432 PGUSER=postgres PGPASSWORD=scratch \
  schema/create_schema.sh

# Apply the proposed up migration
PGHOST=localhost PGPORT=55432 PGUSER=postgres PGPASSWORD=scratch \
  psql -f <proposed-up.sql>

# Apply the proposed down migration (verify reversibility)
PGHOST=localhost PGPORT=55432 PGUSER=postgres PGPASSWORD=scratch \
  psql -f <proposed-down.sql>

# Re-apply up (verify idempotency-after-down)
PGHOST=localhost PGPORT=55432 PGUSER=postgres PGPASSWORD=scratch \
  psql -f <proposed-up.sql>

# Tear down
docker rm -f schemagen-scratch
```

If any step fails, fix the migration and re-run. Do not propose a migration that has not round-tripped up → down → up successfully on a scratch DB.

For large / destructive migrations, the scratch validation confirms the SQL parses and executes — it does NOT confirm the migration is safe under production load. State that distinction in the summary.

### Phase 6 — Propose or write
Produce the final output:

- The two migration files (paths + full content).
- The ripple table.
- The scratch validation result.
- The exact `/codegen` invocations needed to regenerate affected components.
- For destructive changes, an explicit "DESTRUCTIVE — confirm before writing" prompt.

In `mode: propose`: stop. The user reviews and approves.

In `mode: write`: for each destructive change, ask for explicit approval before writing. Write the migration files. Do not edit spec or code files directly — the ripple table tells the user (or the next skill invocation) what to edit.

## Output file naming

```
schema/v1/migrations/<YYYYMMDDHHMMSS>_<name>.up.sql
schema/v1/migrations/<YYYYMMDDHHMMSS>_<name>.down.sql
```

Or match the project's existing migrations directory — inspect it before writing. This repo currently stores schema as SQL files under `schema/v1/{tables,funcs,...}/` rather than numbered migrations; confirm the destination with the user if the migrations pattern is not already established.

## References (load on demand)

- `references/change-types.md` — classification rules for schema change types. Load at Phase 1.
- `references/migration-templates.md` — `up.sql` / `down.sql` templates per change type. Load at Phase 3.
- `references/downstream-updates.md` — where ripple effects land (specs, code, tests). Load at Phase 4.
- `references/scratch-validation.md` — end-to-end scratch DB flow. Load at Phase 5.

## What schemagen does not do

- Does not write application code (use `/codegen` for the regeneration the ripple table points to).
- Does not edit spec files directly (surface the required edits; human or specgen owns the write).
- Does not run migrations against non-scratch databases — never against staging, never against production.
- Does not reorganise the existing schema layout.
