# SQL public surface diff

Scope: views and functions under `schema/v1/views/` and `schema/v1/funcs/` (adapt to the project's layout). Tables are diffed by schemagen, not here.

## Additive

- New view.
- New function.
- New column in a view's SELECT list appended after existing columns.
- New optional function parameter with a default value appended after existing parameters.
- New overload of an existing function (different signature).

## Deprecation

- Comment block `-- DEPRECATED: <reason>` directly above the definition.
- Renamed with a wrapper view/function retained that forwards to the new name.

## Breaking

### Views
- View removed.
- View renamed.
- Column removed.
- Column renamed.
- Column type narrowed.
- Column order changed (breaks consumers relying on positional access).
- Underlying filter tightened so fewer rows are returned.
- SECURITY INVOKER changed to SECURITY DEFINER or vice versa (permission model change).

### Functions
- Function removed.
- Function renamed.
- Parameter added (without default), removed, or reordered.
- Parameter type narrowed.
- Return type changed (scalar → table, or different scalar).
- `LANGUAGE` changed (SQL → plpgsql rarely breaking, but side-effect semantics may shift — flag).
- `VOLATILITY` changed (`VOLATILE` → `STABLE` → `IMMUTABLE` affects planner caching — flag).
- `SECURITY DEFINER` / `INVOKER` swap.

### Triggers on public tables

If the project exposes tables and consumers rely on trigger behaviour, add/remove/modify of triggers is breaking. Flag but note: triggers are usually an implementation detail.

## Ambiguous

- View column renamed in-place (`SELECT foo AS bar` → `SELECT foo AS baz`): breaking for column-name consumers, invisible to `SELECT *` consumers.
- Function returning a composite type with a new field appended: breaking for strict binding consumers.

## Evidence format

```
- Object: view analytics.user_activity
- Change: column `last_seen_at` type changed from TIMESTAMPTZ to DATE
- Base: schema/v1/views/user_activity.sql:12
- Head: schema/v1/views/user_activity.sql:12
- Classification: breaking
```
