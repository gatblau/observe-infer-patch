# Phase C — Test Generation & Verification

Every file in Phase B that contains logic gets a corresponding `_test.go` file. Test files live in the same package (not `_test` external package) unless the spec specifies black-box testing.

## C1 — Unit tests

- One `Test<Function>_<Scenario>` per Gherkin scenario in the spec's Acceptance Criteria.
- One `Test<Function>_Error_<Condition>` per row in the spec's Error Table.
- Table-driven tests (`[]struct{ name string; ... }`) whenever ≥3 cases exercise the same code path.
- Mock external dependencies using interfaces defined in Phase B (repository, clients, publishers). Hand-written mocks in `testutil_test.go`, not generated mocks, unless the project already uses a mocking library.
- Validation tests: every field constraint declared in the spec's Data Model gets at least one positive and one negative case.

## C2 — Integration tests

For components that touch DB, NATS, or MinIO, produce tests with build tag `integration` using testcontainers-go:

```go
//go:build integration

package component

// Each test:
//   1. Starts required containers via testcontainers-go
//   2. Runs migrations from migrations/
//   3. Executes the scenario against the real dependency
//   4. Tears down containers via t.Cleanup
```

Produce at minimum one happy-path and one error-path integration test, **fully implemented** — not stubs, not `t.Skip`.

## C3 — Test helpers

If (and only if) the component needs shared fixtures, factories, or mock builders, put them in `internal/domain/component/testutil_test.go`. Do not create helpers speculatively.

## C4 — Blocking verification

After generating all code and tests, run the following sequence. Code generation is **not complete** until every step passes.

```bash
# 1. Compile — must succeed with no errors
go build ./internal/domain/component/...

# 2. Unit tests — must pass with ≥70% total coverage
go test ./internal/domain/component/... -v -count=1 -cover -coverprofile=coverage.out
go tool cover -func=coverage.out | awk '/^total:/ { print $3 }'

# 3. Lint — must pass with no errors
golangci-lint run ./internal/domain/component/...
```

If any step fails:
1. Identify the root cause (not the symptom).
2. Fix the implementation or test in Phase B/C.
3. Re-run the entire verification sequence.
4. Do not deliver the response with any step failing.

## What Phase C is not

- Not benchmark tests unless the spec defines a performance-critical path with explicit latency targets.
- Not fuzz tests unless the spec requires them.
- Not end-to-end tests that cross multiple components — those are owned by the integration suite orchestrated elsewhere.
