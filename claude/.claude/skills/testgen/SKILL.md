---
name: testgen
description: Retrofit unit and integration tests for existing source code that lacks coverage. Reverse-engineers behaviour from the code (no spec required), identifies uncovered branches and error paths, and generates table-driven tests with mocks. Trigger on phrases like "generate tests for <file>", "retrofit tests", "testgen <package>", "improve coverage of X", or when the user asks to add tests to existing code without a spec. Do NOT trigger on greenfield "generate code + tests from a spec" (that is codegen).
---

# testgen — Test retrofit for existing code

Produce unit and integration tests for code that was written without a spec. Different from `codegen`: there is no spec — behaviour is reverse-engineered from the code itself and any existing tests, then tested against current behaviour (not against a desired contract).

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Arguments

- `target` (required): a file, package path, or directory to cover. E.g. `lib/nats_jwt.go`, `./route`, `internal/domain/user/`.
- `coverage-goal`: default `70` (percent). Minimum acceptable line coverage for the target.
- `mode`: `generate` (default) | `audit-only` (produce coverage gap report; do not write tests).

## Active mode

**Observation (Phase 1–4) → Patch (Phase 5–7).** Phases 1–4 probe toolchain, baseline coverage, and extract behaviour from the code as-written — pure observation. Phase 5–6 write tests; Phase 7 reports. When a generated test fails because current behaviour looks wrong, testgen does NOT switch to fixing the code — it marks `t.Skip` with a reason and routes to `/rca`. That stop-and-surface step is exactly the patch-mode contract for unverified assumptions.

## Hard rule

**Testgen tests current behaviour, not desired behaviour.** If the code has a bug, the test will encode the bug. Testgen surfaces the coverage gap; it does not decide whether the code is correct. Fixing bugs is a separate `/rca` → `/breakdown` → `/apply` cycle.

State this explicitly in the final summary. Users must understand what they are signing off on.

## Process

### Phase 1 — Toolchain probe
Run `go version` (or the language toolchain for the target). Verify a test runner and coverage tool are available. Load `references/language-toolchain.md` for the exact commands per language.

If the target's language is not supported, stop and tell the user which languages are supported.

### Phase 2 — Coverage baseline
Run the project's existing test suite against the target and capture current coverage:

```bash
go test ./<target>/... -cover -coverprofile=coverage.baseline.out
go tool cover -func=coverage.baseline.out
```

Record per-function coverage. This is the "before" number — every generated test must move it toward `coverage-goal`.

### Phase 3 — Behaviour extraction
Load `references/coverage-probes.md`. For each function in the target:

1. Extract the signature and exported/unexported status.
2. Identify every branch (`if`, `switch`, early return, error-return path).
3. Identify every external call (DB, HTTP, NATS, filesystem, time.Now, random). These are mocking boundaries.
4. Identify every validation or precondition check.
5. Record the set of observable outcomes (return values + side effects).

The output of this phase is a per-function behaviour table. Do not guess — read the code. If a function's behaviour cannot be determined by reading (e.g. depends on reflection or runtime configuration), flag it and skip test generation for that function.

### Phase 4 — Gap identification
Cross-reference Phase 3's branches against Phase 2's coverage. Every uncovered branch is a gap. Every untested error path is a gap. Rank gaps:

- **Critical** — untested error path on a public function; untested branch that writes to DB / publishes a message / mutates external state.
- **Important** — untested branch on a public function that only reads.
- **Minor** — untested branch on an unexported helper.

Generate tests in that order.

### Phase 5 — Test generation
Load `references/test-patterns.md`. For each gap, generate a test using the language's idiomatic pattern:

- Table-driven tests when ≥3 cases share a path.
- Hand-written mocks for interfaces (do not invent new mocking frameworks the project does not use).
- Integration tests with build tag `integration` for DB/NATS/MinIO-touching functions, using the project's existing testcontainers-go setup where available.
- One happy path + every error-path case at minimum.

Respect the project's test conventions: same package (white-box) unless existing tests in the target are `_test` (black-box). Match existing naming: `TestFunctionName_Scenario` is standard in Go.

Do not modify the source code under test. Do not add new dependencies to `go.mod` unless the user explicitly approves.

### Phase 6 — Verification
Run the generated tests and re-measure coverage:

```bash
go test ./<target>/... -v -count=1 -cover -coverprofile=coverage.after.out
go tool cover -func=coverage.after.out
```

If coverage has not reached `coverage-goal`, either:
- Add more tests for the remaining critical/important gaps, or
- Stop and report the remaining gaps honestly — do not pad with trivial tests to hit a number.

Every generated test must pass on current code. If a test fails because the current behaviour looks wrong, do not "fix" the code — report the failure as a suspected bug, leave the test marked `t.Skip("suspected bug in <function>; verify before enabling")`, and tell the user to run `/rca`.

### Phase 7 — Summary
Report:
- Baseline coverage → post-generation coverage.
- Files added (test files, testdata, helpers).
- Critical gaps closed, remaining open.
- Any tests skipped due to suspected bugs — list each with the function name and the unexpected behaviour observed.
- Explicit reminder: "These tests encode current behaviour. If any encoded behaviour is incorrect, fix the code and update the test — do not read the test as specification."

## Output files

- `<package>/<file>_test.go` for unit tests (same package by default).
- `<package>/<file>_integration_test.go` with `//go:build integration` for integration tests.
- `<package>/testutil_test.go` only if shared fixtures/factories are needed.
- No new source files. No dependency additions.

## References (load on demand)

- `references/language-toolchain.md` — per-language test + coverage commands. Load at Phase 1.
- `references/coverage-probes.md` — how to identify branches, calls, and outcomes per language. Load at Phase 3.
- `references/test-patterns.md` — table-driven, mocking, integration, fixture patterns. Load at Phase 5.

## What testgen does not do

- Does not fix bugs in the code under test.
- Does not refactor the code under test.
- Does not generate tests for dead code — flag it for deletion instead.
- Does not add behavioural assertions that are not verifiable from the current code.
- Does not produce documentation (no GoDoc, no README).
