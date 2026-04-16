# Constraints & integrity

Concerns: declared PKs, FKs (and their cascade policies), UNIQUE, NOT NULL, and CHECK — whether the constraint set protects the invariants the domain actually requires.

## Patterns to flag

### FK without explicit cascade policy
- `REFERENCES parent(id)` with no `ON DELETE` / `ON UPDATE`. Default is `NO ACTION` (check at end of statement, fail if rows reference) — often fine, but silent. A reader cannot tell whether `NO ACTION` was the considered choice or the accidental one.
- `INTEGRITY`, severity `low`. Upgrade to `medium` when the child table is log/audit (usually wants `SET NULL` or `CASCADE`) or when the parent is subject to routine deletion.

### Inconsistent cascade policy across a tenant
- One child of `tenant` uses `ON DELETE CASCADE`, another uses `NO ACTION`. Deleting a tenant will fail partway — leaving a half-deleted aggregate.
- `INTEGRITY`, severity `high`.

### `ON DELETE CASCADE` without an index on the FK column
- CASCADE delete will tablescan the child on every parent delete. On large tables, becomes a foot-gun at scale.
- `PERFORMANCE` is the primary label here; cross-reference `indexing-and-performance.md`.

### `NO ACTION` / `RESTRICT` where the domain implies CASCADE
- Child rows that have no meaning without the parent (e.g. `appliance_config` rows tied to a deleted appliance) but the FK is `NO ACTION`. Orphans accumulate; manual cleanup required.
- `INTEGRITY`, severity `medium`.

### Missing `UNIQUE` on a natural key
- Column named `key`, `code`, `slug`, `email` with no `UNIQUE` (or composite `UNIQUE (key, tenant_id)`). Duplicate-key surprises at write time, ambiguous upserts.
- `INTEGRITY`, severity `high`.

### `UNIQUE` on global scope where tenant-scoped is meant
- `UNIQUE (key)` on a tenant-scoped entity. Two tenants can't both use the same `key` — usually not the domain intent.
- `INTEGRITY`/`MODELLING`; prefer `MODELLING` and cross-reference `relational-patterns.md`.

### Missing `NOT NULL` on a column the application treats as required
- FK column (`site_type_id BIGINT`) nullable when the application always assigns it. NULL becomes "not yet set" vs "intentionally unset" vs "bug" — ambiguous forever.
- `INTEGRITY`, severity `medium`.

### Composite UNIQUE on a many-to-many table is missing
- `tenant_user (tenant_id, user_id)` with a surrogate `id` PK but no `UNIQUE(tenant_id, user_id)`. Same (tenant, user) pair can appear twice.
- `INTEGRITY`, severity `high`.

### CHECK constraint absent on an enumerated column
- `status VARCHAR` with application code expecting one of `{active, inactive, suspended}` but no `CHECK (status IN ('active','inactive','suspended'))` and no FK to a status lookup. Stray values accumulate.
- `INTEGRITY`, severity `medium`.

### CHECK constraint expresses a business rule encoded only in code
- Application enforces `price > 0` but DB does not. Any direct SQL write (migration, bug, admin tool) can violate it silently.
- `INTEGRITY`, severity `low`–`medium` depending on the rule's criticality.

### Overlapping / redundant constraints
- Same CHECK expressed twice with different constraint names. Harmless, but a COHERENCE smell — pick one, remove the other.
- `COHERENCE`, severity `info`–`low`.

### `DEFERRABLE` / `INITIALLY DEFERRED` with no justifying case
- `DEFERRABLE INITIALLY DEFERRED` applied without a stated reason (circular FK, large-batch load). Changes failure mode: errors only surface at commit time.
- `INTEGRITY`, severity `low`. Flag for docs, do not downgrade to info.

### FK naming convention drift
- `CONSTRAINT site_tenant_id_fk` in one file, `CONSTRAINT fk_site_tenant` in another. No functional impact; error messages become hostile to greppers.
- `COHERENCE`, severity `info`–`low`.

### Missing PK
- Table with no PK. Postgres replication, change-data-capture, and ORM identity all break or become awkward.
- `INTEGRITY`, severity `high` (usually accidental).

### PK on a column without a sequence/default
- PK column without a default/sequence. Callers must generate the id — either the app does it well (UUIDv7) or poorly (sequential counts with races).
- `INTEGRITY`, severity `low` unless it's demonstrably causing id conflicts, then `high`.

### CASCADE chains deeper than two levels
- CASCADE on A→B, CASCADE on B→C, CASCADE on C→D. Deleting A wipes four tables. Usually correct, but worth calling out for rollback planning.
- `INTEGRITY`, severity `info`–`low`. Add a "rollback hazard" note in the remediation (this is destructive by design).

### Trigger-enforced invariant with no fallback constraint
- Trigger enforces "total must equal sum of children". If the trigger is ever disabled (ALTER TABLE … DISABLE TRIGGER), the invariant silently breaks. Pair with a CHECK where feasible.
- `INTEGRITY`, severity `medium`.

## Safe patterns (not findings)

```sql
CONSTRAINT site_id_pk PRIMARY KEY (id),
CONSTRAINT site_tenant_id_fk FOREIGN KEY (tenant_id)
    REFERENCES tenant (id)
    ON UPDATE NO ACTION
    ON DELETE CASCADE,
UNIQUE (key, tenant_id)
```

- Explicit cascade (`CASCADE` / `SET NULL` / `NO ACTION`) stated every time.
- Natural keys always have a scoped UNIQUE on entity tables.
- CHECK constraints mirror every hard domain rule the app relies on.

## Evidence required

- The full constraint block from the table DDL.
- For cascade-consistency findings, every child table of the shared parent, with their cascade policies side-by-side.
- For missing-UNIQUE findings, the call sites (in functions or the app) that perform lookups on the would-be natural key — shows intent.

## Severity cues

| Cue | Suggested severity |
|---|---|
| Missing PK | High |
| Missing UNIQUE on a natural key | High |
| Missing UNIQUE on a junction (m:n) table | High |
| Inconsistent cascade policies across siblings of the same parent | High |
| `ON DELETE CASCADE` without supporting index | High (cross-label PERFORMANCE) |
| FK with no explicit `ON DELETE` | Low–Medium |
| Missing `NOT NULL` on application-required column | Medium |
| CHECK missing on enumerated column | Medium |
| Business-rule CHECK missing | Low–Medium |
| CASCADE chain depth ≥3 | Info–Low |
