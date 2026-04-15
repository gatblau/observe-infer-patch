# Spec ↔ Code checks

Inspect-only comparisons between Phase 3 / Phase 4 specs and source files. Every aspect uses the highest available extraction tier (see `extraction-tiers.md`). Language-agnostic where possible; tier-4 toolchain used only when installed and when T3 is insufficient.

## Resolving the file(s) for a component

1. Read the component's Phase 3 spec — look for the `**File:**` field in the spec header.
2. If present and the file exists → extract from that file.
3. If present and the file does **not** exist → emit `SCOPE-GAP` (spec orphan) and stop aspect checks for this component.
4. If absent → fall back to name-match: search the source tree for files whose basename (without extension) matches the component name in snake_case, kebab-case, or PascalCase. Record the match used.

For cross-cutting concerns (Phase 4), a component may legitimately touch many files. Use the Phase 4 spec's "Integration Points" or equivalent section to enumerate the files to inspect.

## Aspect S1 — Public interface / signatures

**Rule:** Every entry in the spec's Public Interface section has a matching definition in code.

**Extraction (spec side):** function/method names, parameter lists, return types from the spec's Public Interface markdown.

**Extraction (code side):** T3 tree-sitter query for `function_declaration` / `method_declaration` / equivalent per grammar. T4 for final signature adjudication if available and ambiguous.

**Finding comparisons:**
- Name mismatch → drift (label per likely-source heuristic).
- Parameter name or type mismatch → drift.
- Return type mismatch → drift.
- Function in code not in spec → `SCOPE-GAP` (code orphan) unless it is unexported / private — in which case ignore it (implementation detail).
- Function in spec not in code → drift.

## Aspect S2 — HTTP routes

**Rule:** Every route declared in the spec's Public Interface / API contract is registered in code with the same method, path, and status codes.

**Extraction (spec side):** HTTP method + path + response schema + status codes from the Public Interface section.

**Extraction (code side):** T2 regex over common router idioms (`app.Get("/…")`, `router.post("/…")`, `@GetMapping("/…")`, `@app.route("/…")`, FastAPI `@router.get("/…")`, Echo `e.GET("/…")`). T3 tree-sitter for languages where decorator/annotation nodes are cleaner.

**Finding comparisons:**
- Path mismatch (including trailing-slash differences) → drift.
- Method mismatch → drift.
- Route in code not in spec → `SCOPE-GAP` (code orphan).

## Aspect S3 — JSON / serialisation field names

**Rule:** Field names on request/response schemas in the spec match the serialisation tags / alias declarations in code.

**Extraction (spec side):** schema tables or struct examples in the Public Interface section.

**Extraction (code side):**
- Go: struct tag `json:"…"` via T3 grammar `field_declaration` → `tag`.
- TypeScript: interface/class properties via T3.
- Python: Pydantic `Field(..., alias=…)` or dataclass field names via T3/T4.
- Rust: `#[serde(rename = "…")]` attributes via T3.
- Java: `@JsonProperty("…")` via T3.

**Finding:** any name mismatch — including casing differences — is a finding. Do not normalise casing.

## Aspect S4 — Error codes

**Rule:** Every code string in the spec's Error Table appears as a string literal in the corresponding source file, at an error-construction site.

**Extraction (spec side):** Code column of the Error Table.

**Extraction (code side):** T2 grep for the exact string literal inside the component's files, anchored near error construction (within 5 lines of `error`, `err`, `Error`, `raise`, `throw`).

**Finding:**
- Spec code not found in code → drift.
- Error string in code not in spec → `SCOPE-GAP` (code orphan).

## Aspect S5 — Environment variables

**Rule:** Every env var in Phase 2D Configuration table is read somewhere in the code; every env var read in code is listed in Phase 2D.

**Extraction (spec side):** Variable column of Phase 2D table.

**Extraction (code side):** T2 grep across all source files:
- Go: `os.Getenv("X")`, `os.LookupEnv("X")`, viper keys.
- Node/TS: `process.env.X`, `process.env["X"]`.
- Python: `os.environ["X"]`, `os.getenv("X")`, `settings.X` (if using pydantic-settings).
- Rust: `std::env::var("X")`, `env!("X")`.
- Java: `System.getenv("X")`.
- Ruby: `ENV["X"]`, `ENV.fetch("X")`.

**Finding:**
- Env var in spec not in code → `SCOPE-GAP` (spec orphan) — may be legitimate if the component hasn't been built yet.
- Env var in code not in spec → `SCOPE-GAP` (code orphan) or `REALITY-DRIFT`.

## Aspect S6 — Metric names

**Rule:** Every metric name listed in the spec's Observability section (or Phase 4 Metrics spec) is registered in code.

**Extraction (spec side):** metric names explicitly listed.

**Extraction (code side):** T2 grep near metric-registration idioms (`NewCounter`, `NewHistogram`, `NewGauge`, `prometheus_client`, OTel `meter.create_counter`, `metrics.NewInt64Counter`). Extract the string literal argument.

**Finding:** name mismatch (including label set) → drift.

## Aspect S7 — Middleware / cross-cutting references

**Rule:** If Phase 4 specifies that a middleware (auth, rate-limit, correlation-ID, CORS) must be applied to endpoint X, the handler registration for endpoint X in code must reference that middleware identifier by name.

**Extraction (spec side):** Phase 4 spec names the middleware identifier (e.g. `AuthMiddleware`, `RequireJWT`, `withRateLimit`). The Phase 3 component spec lists which endpoints require it.

**Extraction (code side):** T2 grep — does the handler registration line for the relevant route contain the middleware identifier as a call or chain element?

**Finding:** missing reference → drift. This is a structural check, not a behavioural one — the skill does not verify the middleware actually works.

## Aspect S8 — Data model ↔ migrations

**Rule:** Every table, column, type, and NOT NULL constraint in a Phase 3 component's Data Model section exists in `migrations/*.sql`. Every migration file corresponds to a Data Model section.

**Extraction (spec side):** DDL blocks inside Phase 3 Data Model sections — table name, column name, column type, nullability, primary/foreign key declarations.

**Extraction (code side):** T2 parse of `migrations/*.sql`:
- `CREATE TABLE [IF NOT EXISTS] <name> (…)` — extract name.
- Inside parentheses, per-line: `<col_name> <type> [NOT NULL] [DEFAULT …] [REFERENCES …]`.
- `ALTER TABLE <name> ADD COLUMN …` etc — track cumulatively by timestamp order.

**Finding:**
- Table/column in spec not in migrations → drift.
- Table/column in migrations not in spec → `SCOPE-GAP` (code orphan) or `REALITY-DRIFT` if migration file is newer than spec.
- Type mismatch (e.g. spec says `TEXT`, migration says `VARCHAR(255)`) → drift.
- NOT NULL mismatch → drift. This one is load-bearing for data correctness; do not downgrade.

## Aspect S9 — Migration file-pair integrity

**Rule:** Every `migrations/<ts>_<name>.up.sql` has a matching `.down.sql` with the same `<ts>_<name>` prefix.

**Extraction:** T1 filesystem — list files, pair by prefix.

**Finding:** unpaired up or down → `SCOPE-GAP`. Not a classical drift, but always reported.

## Aspect S10 — NATS / message subjects (if present)

**Rule:** Every subject declared in the spec's Public Interface (for consumers) or Events Produced section (for publishers) is referenced as a string literal in code near the subscribe/publish call.

**Extraction:** T2 grep — subject strings often contain `.` (e.g. `tenant.*.site.*.appliance.*.command.accepted`). Anchor on nearby `Subscribe`, `Publish`, `nats.Msg`, `js.Subscribe`, `jetstream.Consume`.

**Finding:** subject mismatch → drift. Wildcard-vs-literal mismatch → drift (spec says `tenant.*.site.*.event`, code subscribes to `tenant.acme.site.lab.event` — report as drift; the human decides).

## Filtering implementation details

Do **not** flag as drift:
- Unexported / private symbols that have no counterpart in the spec (they are implementation detail by design; R13).
- Test files (paths under `**/testdata/`, `**/fixtures/`, files ending `_test.*`, `*.spec.ts`, `test_*.py`).
- Generated files (see `extraction-tiers.md` always-excluded paths).

## Confidence reporting

Every finding records its tier. In the report:
- T1/T4 findings → "high confidence".
- T3 findings → "structural confidence".
- T2 findings → "regex-confidence — verify manually".

Users should treat T2 findings as prompts for human inspection, not definitive drift.
