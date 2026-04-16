# PL/pgSQL functions

Concerns: function correctness, volatility labelling, exception handling, dynamic SQL hygiene, `SECURITY DEFINER` safety, and upsert race conditions.

## Patterns to flag

### `SECURITY DEFINER` without `SET search_path`
- Function declared `SECURITY DEFINER` (runs with the owner's privileges) without `SET search_path = public` (or similar). Attackers or careless callers can plant malicious objects in a schema earlier on `search_path` and have them shadow real ones.
- `SECURITY`, severity `high` whenever `SECURITY DEFINER` is used.
- Safe pattern: `SECURITY DEFINER SET search_path = public, pg_temp` (always include `pg_temp` last).

### Dynamic SQL with string concatenation on user input
- `EXECUTE 'SELECT * FROM t WHERE x = ''' || user_input || '''';` — classic injection.
- `EXECUTE format('... %s ...', user_input)` — `%s` inserts the raw value without quoting. Use `%L` (literal) for values or `%I` (identifier) for names.
- `SECURITY`, severity `critical` if the function is callable from app code; `high` if caller is admin-only.

### Upsert via IF-EXISTS branch instead of `INSERT ... ON CONFLICT`
- Function reads `SELECT id FROM t WHERE key = $1`, branches on NULL: INSERT or UPDATE. Two separate statements → race window where two concurrent calls both see NULL and both INSERT, then one fails on the UNIQUE.
- `CORRECTNESS`, severity `medium`–`high` depending on write concurrency.
- Safe pattern: `INSERT ... VALUES (...) ON CONFLICT (key, tenant_id) DO UPDATE SET ...` — atomic. Returns the row id via `RETURNING`.

### `currval(seq)` after a multi-row `INSERT`
- `INSERT INTO t ... VALUES (...), (...);` followed by `RETURN QUERY SELECT currval('t_id_seq');`. `currval` returns only the *last* value assigned — callers think they got the id of one specific row but may have inserted many.
- Even for single-row inserts `currval` is session-correct but fragile: any other INSERT on the same sequence in the function body moves it.
- `CORRECTNESS`, severity `medium`. Prefer `INSERT ... RETURNING id` bound into a variable or a `RETURN QUERY INSERT ... RETURNING id, 'I'::CHAR`.

### `RAISE EXCEPTION` without `ERRCODE`
- `RAISE EXCEPTION 'User does not have permission';` — the client sees SQLSTATE `P0001` (raise_exception), indistinguishable from any other user-raised error.
- `CORRECTNESS`, severity `low`–`medium`. Prefer `RAISE EXCEPTION 'message' USING ERRCODE = '28000'` (or a project-specific ERRCODE convention) so app code can map to HTTP statuses.

### `EXCEPTION WHEN OTHERS THEN NULL;` (or equivalent)
- Swallows every error, including out-of-disk, lock timeout, constraint violation. The caller thinks the function succeeded.
- `CORRECTNESS`, severity `high`.

### `EXCEPTION WHEN OTHERS THEN ... RAISE ...`
- Catching `OTHERS` to log and re-raise is legitimate but costly: in PL/pgSQL, any block with an EXCEPTION clause creates an implicit subtransaction (SAVEPOINT) on every entry. Hot-path functions pay this repeatedly.
- `PERFORMANCE`, severity `low`. Flag only on hot-path functions.

### Volatility label mismatch
- Function body only reads (`SELECT` only, no side effects), declared `VOLATILE`. Planner can't cache results; every call recomputes.
- Function body writes but declared `STABLE` or `IMMUTABLE`. Planner may skip calls — wrong results.
- `CORRECTNESS` on over-claimed purity; `PERFORMANCE` on under-claimed. Severity `medium` for correctness, `low` for performance.
- Safe: writes → `VOLATILE`; reads same-tx → `STABLE`; pure computation on inputs → `IMMUTABLE`.

### `STRICT` not applied where it should be
- Function whose body would error on NULL input, not declared `STRICT`. Returning NULL early for NULL input is the usual contract — declare `STRICT` to let the planner short-circuit.
- `PERFORMANCE`, severity `low`.

### `PARALLEL SAFE` / `PARALLEL RESTRICTED` / `PARALLEL UNSAFE`
- Function unlabelled. Postgres defaults to `PARALLEL UNSAFE` — parallel query plans skipped. For read-only pure functions, label `PARALLEL SAFE`.
- `PERFORMANCE`, severity `low`.

### Return-type mismatch with caller
- Function returns `TABLE(id BIGINT, action CHAR)` but callers in the codebase `SELECT * FROM fn(...) AS (id BIGINT, name TEXT)` — silent column misassignment.
- `CORRECTNESS`, severity `high`. Confidence `regex-confidence` — verify by matching each caller's column list.

### Non-idempotent schema operation inside `DO` or function
- `DO $$ BEGIN CREATE TABLE t (...); END $$;` without `IF NOT EXISTS` guard. Re-running the migration fails.
- In this repo the convention is `IF NOT EXISTS(SELECT relname FROM pg_class WHERE relname = 't') THEN ...` — flag deviations.
- `CORRECTNESS`, severity `medium` (migration re-run hazard).

### Inline permission checks repeated across functions
- Every writer function calls `has_user_permission(user, tenant, 'ACTION-NAME')` inline, with the action name as a string literal. Change the permission model → touch every function.
- `COHERENCE`/`MODELLING`; prefer `COHERENCE`, severity `low`–`medium`.
- Safe pattern: wrap the check in a helper (`assert_permission(...)` that RAISEs on failure) so the check is one line per function.

### Permission check after the read but before the write
- `SELECT id INTO ...; IF has_permission(...) THEN INSERT ...` — the read happens even for unauthorised callers, leaking existence/absence via timing or log side effects.
- `SECURITY`, severity `low`–`medium`.

### Function modifies another tenant's rows
- Function accepts `tenant_key_param` but the UPDATE/DELETE inside uses only a surrogate id without re-checking the tenant. Authorisation sites rely on the caller always passing consistent params.
- `SECURITY`, severity `high`.

### Unused or orphan function
- Function declared in `funcs/*.sql` but not referenced by any caller (grep against app code + other funcs + views). Still dropped in `dropfxs.sql` — adds migration weight.
- `COHERENCE`, severity `low`.

### Function missing from `dropfxs.sql`
- Function declared in `funcs/` but not listed in `dropfxs.sql`. On upgrade, the function survives as a zombie after the drop-and-recreate cycle — signatures may even mismatch.
- `CORRECTNESS`, severity `medium`.

### Very long function bodies
- Function body >200 lines performing multiple distinct responsibilities (CRUD + permission + logging + notification). Hard to audit, hard to change safely.
- `MODELLING` / `COHERENCE`; prefer `COHERENCE`, severity `low`.

### Function calling another function that may recurse
- Mutual recursion between two PL/pgSQL functions with no depth bound. Stack exhaustion at runtime; not visible at create time.
- `CORRECTNESS`, severity `medium`.

### `SELECT INTO` with no `STRICT` where exactly-one-row is expected
- `SELECT id INTO v FROM t WHERE key = $1;` returns silently if 0 rows (v = NULL) or if >1 rows (v = first). Both hide bugs.
- `CORRECTNESS`, severity `medium`. Use `SELECT INTO STRICT` to raise `NO_DATA_FOUND` / `TOO_MANY_ROWS`.

## Safe patterns (not findings)

```sql
CREATE OR REPLACE FUNCTION upsert_site(
    site_key VARCHAR, tenant_key VARCHAR, ...
)
    RETURNS TABLE (id BIGINT, action CHAR)
    LANGUAGE plpgsql
    SECURITY DEFINER
    SET search_path = public, pg_temp
    VOLATILE
    PARALLEL UNSAFE
AS $$
BEGIN
    PERFORM assert_permission(current_user, tenant_key, 'UPSERT-SITE');
    RETURN QUERY
      INSERT INTO site (key, tenant_id, ...)
      VALUES (site_key, tenant_id_for(tenant_key), ...)
      ON CONFLICT (key, tenant_id) DO UPDATE SET ...
      RETURNING site.id, CASE WHEN xmax = 0 THEN 'I' ELSE 'U' END::CHAR;
END;
$$;
```

## Evidence required

- The full function signature (language, security, search_path, volatility).
- The function body — enough of it to show the critique point (permission check placement, dynamic SQL, upsert shape).
- The callers of the function (grepped against app code and other SQL files).

## Severity cues

| Cue | Suggested severity |
|---|---|
| Dynamic SQL concatenating user input | Critical |
| `SECURITY DEFINER` without `SET search_path` | High |
| `EXCEPTION WHEN OTHERS THEN NULL` | High |
| Function mutates across tenants without re-checking tenant | High |
| IF-EXISTS upsert branch (race window) | Medium–High |
| `currval(seq)` after multi-row insert | Medium |
| `RAISE EXCEPTION` without `ERRCODE` | Low–Medium |
| `SELECT INTO` without `STRICT` where exactly-one expected | Medium |
| Volatility label wrong (read declared VOLATILE) | Low |
| Missing `STRICT` on NULL-passthrough function | Low |
| Orphan function / missing from dropfxs | Low–Medium |
