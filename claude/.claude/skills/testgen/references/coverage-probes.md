# Behaviour extraction per function

For each function in the target, produce a row in the behaviour table:

```
| Function | Exported? | Branches | External calls | Preconditions | Outcomes |
|---|---|---|---|---|---|
| ProcessRequest | yes | 4 | db.Query, nats.Publish | input non-nil, userID not empty | returns *Response or error |
```

## Branches — what counts

Count each of the following as a distinct branch:

- Every `if` / `else if` arm.
- Every `switch` case (including `default`).
- Every early `return` (including `return err`).
- Every loop termination condition when it affects outcome (e.g. `break` vs natural exhaustion).
- Every `select` case in Go.
- Every `try` / `catch` arm in TS/Python/Java.
- Every `match` arm in Rust.

Do not count:
- Unreachable code (dead branches).
- Trivial guard clauses that only re-return `nil`/zero value — count these under Preconditions, not Branches.

## External calls — mocking boundaries

Any call that leaves the function's language/runtime and hits something non-deterministic. Identify these because they must be mocked in unit tests.

Examples (Go):
- Database: anything on `*sql.DB`, `*pgx.Conn`, `pgxpool.Pool`, `lib.Database` interface methods.
- HTTP: `http.Client.Do`, third-party clients.
- NATS: `nats.Conn.Publish`, `Subscribe`, `JetStreamContext.*`.
- S3 / MinIO: `minio.Client.*`.
- Time: `time.Now`, `time.After` (inject via `clock.Clock` interface or test-hook `var now = time.Now`).
- Random: `rand.*`, `crypto/rand.*`.
- Filesystem: `os.Open`, `os.ReadFile`, `os.Create`.
- Env: `os.Getenv` (these are configuration, not mocking boundaries — prefer injection via constructor).

For each external call, note:
- The interface or concrete type being called.
- Whether the code uses an interface (easy to mock) or a concrete type (requires refactor or test seam).
- If concrete with no seam: flag as "untestable without refactor" and skip.

## Preconditions

Explicit and implicit constraints the function assumes:

- `if x == nil { return err }` → precondition: x non-nil.
- `assert len(s) > 0` → precondition: s non-empty.
- Type-enforced (non-pointer, non-optional) preconditions: note but do not test.
- Documentation-only preconditions ("caller must hold the lock"): note as Assumption — these are test-resistant.

## Outcomes — observable results

For each function, enumerate every distinct outcome:

- Return values, including error values and their types (`errors.Is(err, ErrNotFound)` vs generic).
- Side effects: DB rows inserted/updated/deleted, messages published (with subject), files written, metrics incremented, logs emitted (only if semantically load-bearing).
- Panics (should be rare in library code; if present, note and test).

Each outcome needs at least one test. A function with four branches and three distinct outcomes needs a minimum of four tests — not three.

## Per-language extraction hints

**Go:** use `go/parser` via a short inline script if complex; for most files, reading the source with `Read` is sufficient. Count `if`, `switch`, `for...range with break`, `select`, `defer` (defers with branching effects are rare — usually ignore).

**TypeScript/JavaScript:** branches include `?:` ternaries, `||`/`??` short-circuits when used for control flow, optional chaining where the absent-case is an outcome.

**Python:** branches include `elif`, `try/except/else/finally`, `with` context managers when they swallow exceptions, list/dict/generator comprehensions with conditions.

**Rust:** `match` arms, `if let`, `while let`, `?` operator (each `?` is an implicit early-return branch).

**Ruby/Java/Kotlin/Swift:** follow the same principles; each control-flow keyword is a branch boundary.

## Untestable signals

Skip test generation and report the function as untestable if:

- All dependencies are concrete types with no interface seam and refactoring is out of scope for testgen.
- Function depends on global mutable state set elsewhere in the process.
- Function's behaviour is time-dependent without a clock seam.
- Function uses reflection or runtime code generation to produce its return value.

In all cases, record the reason. Do not silently skip.

## Building the test matrix

A good test matrix for a function with B branches, C external calls, and O outcomes has:

- Minimum tests: `max(B, O)`.
- One happy-path test that exercises the dominant branch.
- One test per error-return path.
- One table-driven test grouping any 3+ input variations of the same path.
- One integration test (if the function touches DB/NATS/MinIO) — happy-path only at first.

Do not exceed `2 * max(B, O)` unless the function has complex combinations genuinely worth enumerating. Over-testing simple functions is anti-value.
