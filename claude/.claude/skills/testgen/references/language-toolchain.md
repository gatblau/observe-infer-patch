# Test + coverage commands per language

Use the commands listed here exactly — do not invent variations. If a project uses a different runner (e.g. `gotestsum` instead of `go test`), prefer the project's idiom; detect it by inspecting `Makefile`, `package.json`, `pyproject.toml`, or `Cargo.toml`.

## Go

```bash
# Run tests with coverage
go test ./<pkg>/... -v -count=1 -cover -coverprofile=coverage.out

# Per-function coverage
go tool cover -func=coverage.out

# HTML report (for human inspection only — not part of automation)
go tool cover -html=coverage.out -o coverage.html

# Integration tests (build-tag gated)
go test ./<pkg>/... -tags=integration -v -count=1
```

Total coverage line from `go tool cover -func`:
```
total: (statements) 78.4%
```
Parse the percentage from the last column.

## TypeScript / JavaScript (node)

```bash
# Jest
npx jest --coverage <path>

# Vitest
npx vitest run --coverage <path>

# c8 (generic)
npx c8 --reporter=text node <entry>
```

Coverage summary appears in the final "All files" row of the table output.

## Python

```bash
# pytest + coverage
pytest --cov=<pkg> --cov-report=term-missing <tests-dir>

# Unittest + coverage
coverage run -m unittest discover && coverage report -m
```

The last row of `coverage report` shows totals.

## Rust

```bash
# Needs cargo-llvm-cov or cargo-tarpaulin
cargo llvm-cov --package <crate> --summary-only
# or
cargo tarpaulin --package <crate>
```

## Ruby

```bash
# SimpleCov is usually wired via spec_helper.rb
bundle exec rspec <path>
# Coverage report written to coverage/ by SimpleCov post-run hook
```

## Java / Kotlin

```bash
# JaCoCo via Gradle or Maven
./gradlew test jacocoTestReport
# Report at build/reports/jacoco/test/html/index.html
```

## Detecting project idioms

Before running the default command above, inspect:
- `Makefile` — targets named `test`, `cover`, `coverage` often wrap the right invocation with flags the project needs.
- `package.json` scripts — `test`, `test:coverage`.
- `pyproject.toml` `[tool.pytest.ini_options]` — config hints.
- `Cargo.toml` — workspace vs package layout affects `--package` flag.

Prefer the project idiom if present; fall back to the default command above.

## Parsing output

Never grep the raw output with brittle regex. Instead:

- Go: `go tool cover -func=coverage.out | awk '/^total:/ { print $3 }'` → `78.4%`
- Node: Jest/Vitest emit a `coverage-summary.json` — parse that, not the terminal output.
- Python: `coverage json && jq .totals.percent_covered coverage.json`.
- Rust: `cargo llvm-cov --json` then `jq .data[0].totals.lines.percent`.

If a structured output format is available, prefer it over text scraping.
