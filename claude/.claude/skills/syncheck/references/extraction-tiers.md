# Extraction tiers

Four tiers of contract extraction, probed in order. Use the highest available tier for each file. Tag every finding with the tier that produced it so report readers know the confidence level.

## Tier probe (run once at Step 1)

```bash
command -v tree-sitter  # T3 available if present
command -v go           # T4-go available if present
command -v tsc          # T4-ts available if present
command -v python3      # T4-py available if present
command -v cargo        # T4-rust available if present
command -v ruby         # T4-rb available if present
```

T1 (filesystem) and T2 (regex) are always available. Record the full probe result in the report header.

## Tier 1 — Filesystem

**Capabilities:** file existence at declared paths, directory layout, file-pair integrity (migrations up/down), size/empty checks.

**Mechanism:** `Glob`, `Read` existence checks.

**Confidence:** high for existence/layout questions; not applicable to content questions.

**Use for:**
- Spec declares `**File:** <path>` → does the file exist?
- Migration `up.sql` has a matching `down.sql`?
- Phase A6 file plan matches the actual source tree?

## Tier 2 — Regex / text probes

**Capabilities:** extract string literals anchored on language-agnostic syntactic markers.

**Mechanism:** `Grep` with tight patterns. Patterns below are starting points — adapt to the project's idioms.

**Confidence:** medium. Produces false positives on commented-out code and string constants in tests. Always mark findings "regex-confidence — verify manually" in the report.

**Patterns (indicative, language-agnostic anchors):**

| Aspect | Pattern (starting point) | Extracts |
|---|---|---|
| HTTP routes | `(GET\|POST\|PUT\|PATCH\|DELETE)\s*\(?\s*["'`]([^"'`]+)["'`]` | Method + path pairs |
| JSON field names | `["']([a-z_][a-z0-9_]*)["']\s*:` (in response/request construction sites) | Field names |
| Env var reads | `(Getenv\|process\.env\.\|os\.environ\|ENV\[)` | Env var names |
| Error code literals | uppercase snake-case strings near `error`/`Error`/`err` identifiers | Error code strings |
| Metric names | `(NewCounter\|NewHistogram\|NewGauge\|prometheus_client\|metrics\.)` | Metric identifiers |
| SQL table names | `CREATE TABLE (IF NOT EXISTS )?([a-z_]+)` in `migrations/*.sql` | Table names |
| SQL columns | per-table `CREATE TABLE` block parse | Column name, type, nullability |
| Middleware refs | Phase 4 spec names a middleware identifier → grep for it in handler files | Boolean: referenced / not |

## Tier 3 — Tree-sitter (universal AST)

**Capabilities:** structural AST queries across any grammar-supported language. Handles function signatures, struct/class fields, exports, annotations.

**Mechanism:** `tree-sitter parse <file>` then pattern-match against the S-expression output, or use `tree-sitter query` with a language-specific query file.

**Confidence:** high for signature/field extraction. The grammar names the node, so false positives from string constants disappear.

**Supported grammars (install via `tree-sitter init-config && npm i tree-sitter-<lang>` or preinstalled pack):**
- Go: `tree-sitter-go`
- TypeScript/TSX: `tree-sitter-typescript`
- JavaScript: `tree-sitter-javascript`
- Python: `tree-sitter-python`
- Rust: `tree-sitter-rust`
- Java: `tree-sitter-java`
- C#: `tree-sitter-c-sharp`
- Ruby: `tree-sitter-ruby`
- Swift: `tree-sitter-swift`
- Kotlin: `tree-sitter-kotlin`

**Use for:** exported symbols, method signatures, struct/class/record field names and types, decorators/annotations.

## Tier 4 — Native toolchain AST (opt-in)

**Capabilities:** full type information including generics, inferred types, build-tagged files, package/module visibility rules.

**Mechanism (per language):**
- Go: write a short script that uses `go/parser` + `go/types`; invoke via `go run`. Alternative: `go list -json ./...` gives package-level exports.
- TypeScript: `tsc --noEmit --declaration --emitDeclarationOnly` → parse `.d.ts` output for exported signatures.
- Python: `python3 -c "import ast; ..."` one-liner, or `mypy --no-error-summary --no-color-output` for type info.
- Rust: `cargo check --message-format=json` → parse compiler diagnostics for symbol info; `cargo rustc -- -Zunpretty=hir` for HIR.
- Ruby: `ripper` stdlib → S-expressions.

**Confidence:** highest.

**Use for:** final adjudication of signature-level findings where T3 is ambiguous (generics, method receivers, type aliases). Skip for most runs — T3 is sufficient.

## File-extension allowlist (default)

Inspect files with these extensions. Override via `.syncheck.yml` (`include:` / `exclude:` lists) or a block in the repo's `CLAUDE.md` under `## syncheck`:

```
.go .ts .tsx .js .jsx .mjs .cjs
.py .rb .rs .java .kt .scala .cs .swift .m .mm
.sql
.yaml .yml .json  (for OpenAPI, k8s manifests, config)
.proto
.graphql .gql
```

## Always-excluded paths (default)

```
vendor/ node_modules/ target/ dist/ build/ out/
.git/ .idea/ .vscode/ .claude/
docs/                 # design and spec live here; never self-compare
**/generated/ **/*.pb.go **/*_gen.go **/*.generated.*
coverage/ .next/ .nuxt/
```

## Degradation rules

- If T4 is not available for the language, fall back to T3.
- If T3 is not available for the language, fall back to T2 and mark findings "regex-confidence".
- If T2 cannot locate the aspect (e.g. the file is a compiled binary), emit a SCOPE-GAP instead of claiming the aspect is missing.
- Never fabricate a higher tier than was actually used.
