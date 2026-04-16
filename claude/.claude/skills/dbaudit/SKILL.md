---
name: dbaudit
description: Critique a SQL codebase — DDL, constraints, indexes, views, and PL/pgSQL functions — for relational patterns, domain modelling, attribute choices, integrity, indexing/performance, function correctness, and overall coherence. Produces a ranked, classified findings report with recommendations, optionally shaped for direct handoff to `/breakdown`. Trigger on phrases like "dbaudit", "database code audit", "audit the schema", "critique the SQL", "review schema/v1", "review the DDL", or when the user asks for a coherence/performance critique of the SQL under `schema/**` or equivalent. Do NOT trigger for Go call-site concerns (transactions, Rows.Close, Scan), design/spec/code drift (that is `syncheck`), writing a specific migration (that is `schemagen`), or broad security reviews outside SQL (that is `secscan`).
---

# dbaudit — SQL codebase critique

Produce a ranked, classified critique of a SQL codebase: tables, constraints, types, indexes, views, and PL/pgSQL functions. Static inspection only — never runs SQL, never opens a connection, never writes migrations. Output is shaped to be pasted straight into `/breakdown` when the user wants to turn findings into a migration plan (which `/breakdown` then routes through `schemagen` for the DDL author step).

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them. The **SQL & schema safety** layer (`.claude/rules/layer/sql-safety.md`) is always-on here: do not name a table, column, index, trigger, view, constraint, sequence, or function unless it is directly visible in the repo, and surface rollback considerations on anything destructive.

## Scope

Default scope: `schema/**/*.sql` (DDL, PL/pgSQL functions, views, content/i18n seeds). Works against any project layout that groups SQL by concern — table files, function files, view files — such as this repo's `schema/v1/command/{tables,funcs,views,i18n,content}/**`.

**Out of scope:**

| Concern | Use instead |
|---|---|
| Writing a specific migration (`ALTER TABLE …` authoring) | `schemagen` |
| Drift between design / spec / code / schema | `syncheck` |
| Go code that accesses the DB (pool, transactions, scans) | out of scope for this skill |
| Generic security audit (secrets, auth, transport) | `secscan` |
| Test coverage over DB code | `testgen` |
| Turning dbaudit findings into an incremental plan | feed this report into `/breakdown` |

## Arguments

- `scope`: default `schema/**/*.sql`. Accepts a path (`schema/v1/command/tables/locations/`), a single file (`schema/v1/command/funcs/site.sql`), or a concern keyword (`tables`, `constraints`, `indexes`, `views`, `functions`, `coherence`).
- `mode`:
  - `report` (default) — findings only.
  - `recommendations` — findings + a short "Suggested remediation" block per finding (text only — no DDL diffs).
  - `breakdown-ready` — findings + a "Handoff to /breakdown" block pre-formatted for that command (see `references/breakdown-handoff.md`).
- `severity-floor`: default `low`. Suppresses findings below this threshold. One of `info`, `low`, `medium`, `high`, `critical`.

## Active mode

**Observation + Inference.** Observation for evidence gathering (Phase 1–3: file paths, DDL extracts, line numbers). Inference for classification and severity (Phase 4: "worst-case outcome given this schema shape, under what preconditions"). Never transitions to Patch. Even `recommendations` mode emits prose, not DDL — when the user wants the actual migration, `/breakdown` routes to `schemagen`, which is the only author of DDL in this repo's workflow.

## Hard rules

- **Static inspection only.** No `psql`, no `\d`, no connection to any database, no `EXPLAIN`.
- **Never invent identifiers.** Every named table, column, index, constraint, sequence, view, function, or trigger must be directly visible in the repo. Unverified names are flagged "regex-confidence — verify manually"; if the name cannot be confirmed at all, the finding is downgraded to `info` or dropped.
- **Every finding cites file:line.** A claim without a concrete location is not a finding.
- **Severity must be justified.** Every finding states (a) the worst-case outcome, (b) the preconditions for it, (c) why this severity not the adjacent one.
- **Rollback considerations on destructive remediations.** When a suggested remediation is a `DROP`, `ALTER ... DROP COLUMN`, `ALTER ... TYPE`, or a `NOT NULL` backfill, the recommendation text must name the rollback hazard explicitly (data loss, locks held, dependent views/functions). The user and `schemagen` decide how to mitigate — dbaudit only flags.
- **Do not propose DDL diffs.** In `recommendations` mode, remediations are short English sentences naming the target file and the intended change. No `CREATE`/`ALTER`/`DROP` statements in the report. Producing actual DDL is `schemagen`'s job.
- **Respect confidentiality.** If a SQL file contains what looks like a live credential or seed user password, report only file path and line. Never echo the secret.

## Taxonomy (labels)

Every finding carries exactly one label. Pick the primary harm.

| Label | Meaning | Typical examples |
|---|---|---|
| MODELLING | Relational or domain-model flaw: wrong entity boundary, normalisation issue, broken multi-tenancy, missing or duplicated concept. | Entity table missing `tenant_id`; two tables modelling the same status lookup; derived column duplicating truth from another table; log table with a natural key (violates the log-table carve-out). |
| INTEGRITY | Missing or incorrect constraint: FK, UNIQUE, NOT NULL, CHECK, cascade policy. | FK with no `ON DELETE` policy on a child pointing at a parent that cascades; natural-key column without `UNIQUE`; nullable column the domain requires non-null; CHECK absent on an enumerated range. |
| TYPES | Column-type, nullability, or default-value choice that will surprise at scale or corrupt semantics. | `TEXT` vs `VARCHAR(N)` drift across similar columns; `TIMESTAMP` where `TIMESTAMPTZ` is meant; nullable `created_at`; `NUMERIC` stored as `TEXT`. |
| PERFORMANCE | Indexing or shape decision that produces a seq scan / bloat / lock hazard at expected load. | Missing index on a FK column (Postgres does not auto-index FKs); duplicate/redundant indexes; missing index supporting `ON DELETE CASCADE`; view joining unindexed predicates; composite index with low-cardinality leading column. |
| CORRECTNESS | PL/pgSQL or view logic bug that yields wrong results or silent failure. | Upsert with a race window instead of `INSERT ... ON CONFLICT`; function that returns `currval(seq)` after a multi-row insert; `EXCEPTION WHEN OTHERS THEN NULL;` swallowing failures; view referencing a column renamed elsewhere. |
| SECURITY | Dynamic SQL injection in a function, missing `SET search_path` on `SECURITY DEFINER`, PII stored without masking, row-level security absent on a tenant-scoped surface. | `EXECUTE format('...' || user_input)` in a function; `SECURITY DEFINER` without `SET search_path`; plaintext seed password. |
| COHERENCE | Convention drift, dead code, naming/grouping inconsistency, missing `COMMENT ON`, deploy-order hazard. | Singular vs plural table names mixed; FK naming pattern diverging across files; orphan function not in `dropfxs.sql` nor referenced; table referenced before declared in the manifest's deploy order. |

## Severity matrix

| Severity | Criteria |
|---|---|
| Critical | Demonstrable data-loss path, SQL injection in a function reachable from the app, broken multi-tenancy (rows of one tenant reachable via a missing FK/filter), plaintext live credential. |
| High | Missing constraint on a user-visible invariant; missing index on a hot FK (CASCADE DELETE will tablescan); view/function whose logic is visibly wrong on the primary use case. |
| Medium | Latent bug requiring specific inputs or scale; redundant-index bloat on a large table; TYPE mismatch that will bite during DST / precision edge; `SECURITY DEFINER` without `SET search_path` on a function not yet exposed broadly. |
| Low | Coherence/style/defence-in-depth gap with no direct failure path. Duplicate index on a small lookup table; missing `COMMENT ON`; sequence start inconsistent with project convention. |
| Info | Observation with no stated threat, bug, or performance impact. |

Every finding records the reasoning. A finding that cannot distinguish its assigned severity from the next-lower one is downgraded.

## Process

### Phase 1 — Stack and layout probe

Determine, from files only:

- **Engine:** PostgreSQL is the default. Detect variants by `CREATE EXTENSION`, `uuid_generate_v4()`, `POINT`, PostGIS casts, `pgcrypto`, etc. Record the engine in the report header.
- **Layout:** where tables, functions, views, i18n, and seed content live. In this repo the reference layout is `schema/v1/command/{tables,funcs,views,i18n,content}/**` with a `manifest.json` orchestrating deploy order.
- **Deploy-order source of truth:** `manifest.json` / migration list / `create_schema.sh`. Record the file. Later phases cross-reference it for COHERENCE findings.
- **Conventions probe:** idempotency wrappers (`DO $$ ... IF NOT EXISTS ... END $$`), sequence-start convention (e.g. this repo starts app IDs at 1000 to reserve 0–999 for seeds), naming style (`snake_case`, singular table names, FK-name pattern `<child>_<parent>_fk`), timestamp policy (`TIMESTAMPTZ DEFAULT NOW()`).

Record these facts in the report header. They are the baseline against which COHERENCE findings are measured.

### Phase 2 — Object inventory

Enumerate every object the scope implies, with `path:line`:

- **Tables** (`CREATE TABLE …`): columns with types/nullability/defaults, PK, all constraints, all indexes, sequences, `TABLESPACE`/storage options, `COMMENT ON` if present.
- **Views** (`CREATE [OR REPLACE] VIEW …`): referenced tables/columns; whether the view itself is queried by functions.
- **Functions** (`CREATE [OR REPLACE] FUNCTION …`): signature, `LANGUAGE`, `SECURITY DEFINER|INVOKER`, `SET search_path`, `VOLATILE|STABLE|IMMUTABLE`, `STRICT`, return type, exception blocks, dynamic SQL usage.
- **Sequences** declared outside tables.
- **Triggers** and trigger functions (if any).
- **Seed/i18n content** files — scanned for live credentials and for references to tables/columns that must exist.

The inventory is the ground truth for every finding. If an object named in a finding is not in the inventory, the finding is dropped.

### Phase 3 — Concern-driven scan

Run each concern the scope implies. Load the matching reference on demand.

| Concern | Reference | Triggers |
|---|---|---|
| Relational patterns, domain model, multi-tenancy, audit/log tables | `references/relational-patterns.md` | Any `CREATE TABLE`. Always on when scope covers tables. |
| Attributes — types, nullability, defaults, timestamps | `references/attributes-and-types.md` | Any column definition. |
| Constraints & integrity — PK, FK, UNIQUE, CHECK, cascade | `references/constraints-and-integrity.md` | Any `CREATE TABLE`, `ALTER TABLE … ADD CONSTRAINT`, or `REFERENCES`. |
| Indexing & performance | `references/indexing-and-performance.md` | `CREATE INDEX`; FK declarations; view joins; high-volume-looking tables. |
| Views | `references/views.md` | Any `CREATE [OR REPLACE] VIEW`; function bodies that SELECT from a view. |
| PL/pgSQL functions | `references/functions-plpgsql.md` | Any `CREATE [OR REPLACE] FUNCTION`; `DO` blocks. |
| Coherence & conventions | `references/coherence-and-conventions.md` | Always on. Cross-file checks. |

If `scope` is a single concern keyword, run only that reference. Otherwise run all applicable concerns.

### Phase 4 — Classification, severity, confidence

For each finding:

1. Assign exactly one taxonomy label (pick the primary harm).
2. Assign severity using the matrix above. Record the preconditions that justify that severity and the reason it is not the adjacent one.
3. Tag confidence:
   - `high` — objects directly visible and behaviour determinable from the DDL alone.
   - `regex-confidence` — pattern-matched; verify manually (names, join conditions, dynamic-SQL fragments).
   - `inferred` — depends on caller behaviour, runtime data volume, or config not visible in the repo. Cap severity at `medium`.
4. Drop findings below `severity-floor`.

### Phase 5 — Report assembly

Emit in this shape:

```markdown
# dbaudit report

**Scope:** {{scope}}
**Severity floor:** {{floor}}
**Mode:** {{mode}}
**Engine:** {{PostgreSQL N, extensions: ...}}
**Layout:** {{tables=…, funcs=…, views=…, manifest=…}}
**Conventions probed:** {{idempotency wrapper=…, seq start=…, naming=…, timestamp policy=…}}
**Generated:** {{UTC timestamp}}
**Git HEAD:** {{short sha}}

## Summary
- By severity: critical={{n}} high={{n}} medium={{n}} low={{n}} info={{n}}
- By label: MODELLING={{n}} INTEGRITY={{n}} TYPES={{n}} PERFORMANCE={{n}} CORRECTNESS={{n}} SECURITY={{n}} COHERENCE={{n}}

## Findings

### F-001 — {{short title}}  ({{severity}}, {{label}})
- **Location:** `schema/v1/command/tables/.../thing.sql:42`
- **Concern:** {{concern category}}
- **Observation:** {{what the DDL says, 1–2 sentences, no speculation}}
- **Why it matters:** {{worst-case outcome}}
- **Preconditions:** {{load, data shape, caller behaviour required}}
- **Confidence:** {{high | regex-confidence | inferred}}
- **Evidence:**
  ```sql
  {{minimal quoted DDL}}
  ```
- **Suggested remediation** (modes `recommendations` and `breakdown-ready` only):
  {{prose — target file, intended change, rollback hazard if destructive. No DDL.}}

### F-002 — …
```

Sort findings by severity descending, then by label, then by file path. Number them `F-001`, `F-002`, … so the `/breakdown` handoff can reference them stably.

### Phase 6 — /breakdown handoff (`breakdown-ready` mode only)

Load `references/breakdown-handoff.md`. Append a **Handoff to /breakdown** section that names the anchor finding(s), lists verified context, flags open assumptions, gives a ready-to-run `/breakdown` invocation, and includes the diagnosis body for the user to paste. The handoff makes clear that DDL remediations are authored by `schemagen` downstream of `/breakdown`, never directly.

dbaudit does not invoke `/breakdown` itself. It stages the handoff and stops.

## What dbaudit does not do

- Does not execute SQL, connect to a database, or run `EXPLAIN`.
- Does not author migrations — that is `schemagen`'s job.
- Does not modify any SQL file.
- Does not compare against design docs or specs — that is `syncheck`.
- Does not audit the Go code that calls the database.
- Does not invoke `/breakdown` — it only stages the handoff.

## References (load on demand)

- `references/relational-patterns.md` — domain modelling, multi-tenancy, audit/log separation, key strategy.
- `references/attributes-and-types.md` — column types, nullability, defaults, timestamp policy.
- `references/constraints-and-integrity.md` — PK, FK (with cascade), UNIQUE, NOT NULL, CHECK.
- `references/indexing-and-performance.md` — missing/duplicate/redundant indexes, FK indexing, view-join support.
- `references/views.md` — view shape, performance, break-risk, materialisation opportunities.
- `references/functions-plpgsql.md` — PL/pgSQL correctness, volatility labelling, exception handling, dynamic SQL, `SECURITY DEFINER` hygiene.
- `references/coherence-and-conventions.md` — naming, grouping, deploy order, dead code, documentation.
- `references/breakdown-handoff.md` — format of the `/breakdown` handoff block, with routing to `schemagen` for DDL steps.
