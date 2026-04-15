# SQL injection

## Target patterns

String concatenation, `fmt.Sprintf`, f-strings, or template literals building SQL that includes a variable unlikely to be a constant.

### Go
- `fmt.Sprintf("SELECT ... %s ...", userInput)` passed to `db.Query` / `db.Exec` / `pgxpool.Query`.
- `"SELECT ... " + userInput + " ..."` as the query argument.
- `sqlx.DB.Rebind` used without placeholders.

### Python
- f-string SQL: `f"SELECT ... {user_input} ..."` passed to cursor.execute.
- `.format()` or `%` formatting applied to a SQL literal.

### TypeScript / JavaScript
- Template literals with `${var}` passed to raw query methods (`pool.query`, `knex.raw`).
- ORM `raw()` escape hatches with interpolation.

## Safe patterns (not findings)

- Parameterised queries: `db.Query("SELECT ... WHERE id = $1", id)` (Postgres), `?` placeholders (MySQL/SQLite).
- ORM query builders with no `raw()` escape.
- Identifier interpolation (table/column names) is NOT safe via placeholders — flag if user-influenced, safe if from a constant allowlist.

## Severity

- **High:** user-input variable flows directly into a raw query; no allowlist between input and SQL.
- **Medium:** variable comes from config/env (not user input) but still concatenated — defence-in-depth gap.
- **Low:** identifier interpolation from an allowlist-enforced source.

## Evidence required

For each finding show:
- The query construction line.
- The source of the variable (follow one level up — function parameter, request body field, env var).
- Whether validation/sanitisation exists between source and sink.
