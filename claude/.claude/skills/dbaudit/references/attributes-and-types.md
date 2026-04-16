# Attributes — types, nullability, defaults

Concerns: column type choices, nullability decisions, default values, and timestamp policy. Subtle here matters — one bad TYPES choice silently corrupts data; one bad default changes the meaning of every future INSERT.

## Patterns to flag

### Character-type drift across similar columns
- Same logical attribute typed inconsistently across tables: `name VARCHAR(100)` here, `name VARCHAR(255)` there, `name TEXT` elsewhere. Truncation surprises, join pain when joining on mis-typed columns.
- `TYPES`, severity `medium`. `high` if the columns are joined or compared.
- Pick one: the project's convention usually emerges from the first few tables. Flag outliers, not the convention itself.

### `VARCHAR(N)` picked without evidence for N
- `VARCHAR(50)` for a `key` column — why 50? If the domain has no documented upper bound, `TEXT` is strictly better in Postgres (no performance penalty, no truncation bug).
- `VARCHAR(100)` for `created_by` where the value is an email or username — eventually a real name exceeds it.
- `TYPES`, severity `low` unless truncation has demonstrable impact, then `medium`.

### `TIMESTAMP` where `TIMESTAMPTZ` is meant
- Column stores an absolute point in time but the type is `TIMESTAMP` (no time zone). Compares against `NOW()` / `CURRENT_TIMESTAMP` go through an implicit cast using the session's timezone — silent wrong answers across DST, across clients in different zones.
- `TYPES`, severity `high` if used for expiry/scheduling/authorization, `medium` otherwise.
- Safe pattern: `TIMESTAMPTZ DEFAULT NOW()` for all event-time and audit columns.

### Nullable `created_at` / audit columns
- `created_at TIMESTAMPTZ` with no `NOT NULL` and no `DEFAULT NOW()`. Rows can be inserted without an audit stamp.
- `TYPES` + `INTEGRITY`; prefer `INTEGRITY` label since the fix is a constraint. Severity `medium`.

### `DEFAULT NOW()` on a nullable column
- Redundant — the column could just be `NOT NULL DEFAULT NOW()`. Flag as `COHERENCE`, severity `info`, unless nullability is load-bearing (it rarely is on `created_at`).

### Nullable natural-key column
- `key VARCHAR(50)` nullable on an entity table. A NULL natural key means the row has no addressable identity outside its surrogate. Usually a modelling error.
- `TYPES` (nullability dimension), severity `medium`.

### `NUMERIC` precision unspecified
- `NUMERIC` with no precision/scale. Postgres accepts arbitrary precision, but callers lose the contract — a financial column should be `NUMERIC(p, s)` with documented p and s.
- `TYPES`, severity `medium` on financial/quota paths; `low` otherwise.

### `FLOAT` / `DOUBLE PRECISION` for money or counts
- Money/quantity columns stored as `FLOAT`. Rounding errors accumulate.
- `TYPES`, severity `high` on monetary paths, `medium` on quantity counts.

### `TEXT` for structured data
- `TEXT` storing JSON — use `JSONB` (indexable, validated, efficient operators).
- `TEXT` storing an enum value without a lookup table or CHECK — use a lookup table or `CHECK` against the known set.
- `TYPES`, severity `medium`.

### Booleans encoded as integers or strings
- `is_active INT` (0/1), `active VARCHAR` ('Y'/'N'). Postgres has native `BOOLEAN`; prefer it.
- `TYPES`, severity `low`.

### Geographic types without documented semantics
- `location POINT` stores (x, y) pairs but the project uses `POINT(latitude, longitude)` (lat-first, lng-second) in insert functions. PostGIS convention is (longitude, latitude). Functions downstream that assume the other order produce wrong distances.
- `TYPES`, severity `medium`. Flag when the POINT is created in one order but consumed in code that uses the opposite.

### `VARCHAR` storing arrays via delimiters
- `tags VARCHAR(255)` storing `"a,b,c"`. Use `TEXT[]` — indexable, expressive, no split-bug risk on values containing commas.
- `TYPES`, severity `medium`.

### Enumerated column without `CHECK` or FK
- `status VARCHAR` that the application treats as one of a fixed set, but no `CHECK (status IN (...))` and no FK to a status lookup table. Data drift over time — stray values accumulate.
- `TYPES`/`INTEGRITY`; pick `INTEGRITY` since the fix is a constraint.
- Cross-reference `constraints-and-integrity.md`.

### `CHAR(N)` (fixed-width) instead of `VARCHAR`/`TEXT`
- Fixed-width CHAR pads with spaces. Almost always a porting artefact; no good reason in Postgres.
- `TYPES`, severity `low`.

### Defaults that mask intent
- `DEFAULT ''` on a required business attribute. Turns "missing" and "explicitly empty" into the same value.
- `DEFAULT 0` on a FK or status id. Silently binds rows to an arbitrary row (usually row id 0, which may not exist).
- `TYPES`, severity `medium`.

### Mismatch between column default and i18n / seed semantics
- `sort_order INT DEFAULT 0` when the seed assigns `sort_order` starting from 100. New inserts at default land above all seeds — UI reorders unexpectedly.
- `TYPES`, severity `low`.

## Safe patterns (not findings)

- `id BIGINT NOT NULL DEFAULT nextval('thing_id_seq')` paired with a dedicated sequence.
- `tenant_id BIGINT NOT NULL` + FK.
- `key VARCHAR(50) NOT NULL` + `UNIQUE(key, tenant_id)`.
- `created_at TIMESTAMPTZ NOT NULL DEFAULT NOW()`.
- `created_by VARCHAR(100) NOT NULL` (the convention in this repo).
- `location POINT` with POINT order **documented** at the insert call sites.

## Evidence required

- The column declaration with full type, nullability, and default.
- Where the same logical column appears in other tables, for drift findings.
- For default-value surprises, the seed/i18n data or a function body that demonstrates the interaction.

## Severity cues

| Cue | Suggested severity |
|---|---|
| `TIMESTAMP` (not `TIMESTAMPTZ`) on auth/expiry path | High |
| `FLOAT` for money | High |
| Nullable natural-key column on entity | Medium–High |
| Same logical column typed inconsistently across tables joined together | Medium–High |
| `VARCHAR(N)` with N chosen with no evidence, no truncation observed | Low |
| `CHAR(N)` fixed-width | Low |
| Default that masks "missing" vs "empty" | Medium |
