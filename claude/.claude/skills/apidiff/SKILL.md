---
name: apidiff
description: Compare two versions of an API contract (HTTP/REST, gRPC, NATS subjects, Go/TS public package surface) and classify every change as additive, deprecation, or breaking. Produces a semver recommendation, a migration note for consumers, and a changelog fragment. Trigger on phrases like "apidiff", "what changed in the API between X and Y", "is this a breaking change", "generate a changelog for the API", "compare openapi versions", or when the user asks to audit an API change before release. Do NOT trigger on code-level diffs with no public surface (that is a plain git diff) or on drift detection against a spec (that is syncheck).
---

# apidiff — API contract evolution diff

Compare two versions of an API surface and classify every change. Emit a semver recommendation, a consumer-migration note, and a changelog fragment.

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Arguments

- `base` (required): the baseline version. One of: a git ref (`v1.2.0`, `main`, `<sha>`), a file path to a contract (`docs/openapi.v1.yaml`), or a directory.
- `head` (required): the version to compare. Same forms as `base`.
- `surface`: which contract to diff. Defaults to auto-detect across all supported surfaces. One of: `openapi`, `grpc`, `nats`, `go-public`, `ts-public`, `sql` (public views/functions only), `all`.
- `consumers`: optional — path or free-text list of known consumer components/repos. Used to annotate breaking changes with who needs to update.
- `mode`: `report` (default) | `changelog` (emit only the changelog fragment) | `gate` (exit non-zero-style signal if breaking changes found — for CI framing).

## Active mode

**Observation.** Read-only on both `base` and `head`; classification rules per surface are deterministic (additive/deprecation/breaking). Ambiguous cases are surfaced for the user, never auto-resolved. Never transitions to Patch — apidiff does not edit contracts, write changelogs, or bump versions.

## Hard rules

- **Auto-detect, don't assume.** Inspect both `base` and `head` for supported surfaces before picking. If both contain an OpenAPI file but only `head` has gRPC protos, diff each surface independently.
- **Classify every change.** No change is unclassified. The three buckets are exhaustive: additive / deprecation / breaking.
- **Consumers are evidence, not opinion.** A change is "breaking" if it can break a conforming existing consumer. "No one calls it" is not a reason to downgrade severity — surface it and let the user decide.
- **Never modify the base.** apidiff is read-only across both ends. No checkouts that leave dirty working trees; use `git show` / `git archive` to extract `base` if it's a ref.

## Process

### Phase 1 — Resolve `base` and `head`
- Git ref: `git show <ref>:<path>` for single files; `git archive <ref>` piped to a temp dir for trees. Prefer not to check out — leaves HEAD alone.
- File path: read directly.
- Directory: read directly, no git involvement.

Record the exact commit SHAs (or file hashes) in the report so the diff is reproducible.

### Phase 2 — Surface detection
Load `references/surface-detection.md`. For each surface, probe both ends:

| Surface | Signal |
|---|---|
| OpenAPI / Swagger | `openapi.yaml`, `swagger.json`, `docs/swagger.*`, `@swag` annotations in Go |
| gRPC / protobuf | `*.proto` under `proto/`, `api/`, `pkg/proto/` |
| NATS subjects | Code references to subjects of shape `tenant.*.site.*.appliance.*...` — use `syncheck` extraction tiers |
| Go public surface | Exported identifiers in `pkg/`, `lib/`, root module; skip `internal/` |
| TypeScript public surface | `exports` in `package.json`, `index.ts` re-exports |
| SQL public | Views, functions, materialised views in `schema/v1/{views,funcs}/` |

If a surface is present in only one of base/head, that's itself a finding — the whole surface was added or removed.

### Phase 3 — Diff per surface
Load the relevant reference:
- `references/openapi-rules.md`
- `references/grpc-rules.md`
- `references/nats-subject-rules.md`
- `references/go-public-rules.md`
- `references/ts-public-rules.md`
- `references/sql-public-rules.md`

Each reference defines the concrete additive/deprecation/breaking rules for that surface. Do not invent rules; stick to the references.

### Phase 4 — Classify
Every change lands in exactly one bucket:

- **Additive** — new endpoint, new optional field, new enum value in a response, new RPC. Existing consumers unaffected.
- **Deprecation** — existing surface marked deprecated but still functional. Schedule removal in a future breaking change.
- **Breaking** — removed, renamed, type-narrowed, made-required (request), made-optional (response field consumers rely on), changed semantics.

Borderline cases:
- Enum value added in a request → additive for clients, breaking for the server if the server validates (flag and ask).
- Enum value added in a response → breaking for strict consumers, additive for tolerant ones (flag).
- Response field type widened (`int32` → `int64`) → breaking for wire-format consumers (gRPC), potentially safe for JSON (flag).

When truly ambiguous, present both interpretations with the evidence and let the user pick.

### Phase 5 — Semver recommendation
Apply the standard rule:
- Any breaking change → major bump.
- Additive only → minor bump.
- Bug-fix / doc only / deprecation-add only → patch bump.

State the current version if detectable (from `package.json`, `go.mod` build tags, a `VERSION` file, most recent git tag) and the recommendation.

### Phase 6 — Consumer migration note
For every breaking change, write a consumer-facing note:

```markdown
### {{Breaking: description}}

**Before:**
```
{{old signature / schema / subject}}
```

**After:**
```
{{new signature / schema / subject}}
```

**Migration:** {{what consumers must do, concretely}}
```

If `consumers` was provided, annotate each breaking change with "Known consumers affected: X, Y". Otherwise state "Known consumers: not provided — enumerate before release."

### Phase 7 — Output assembly
Produce four sections:

1. **Summary** — counts per surface, semver recommendation, commits/file hashes compared.
2. **Findings by surface** — one section per surface that had changes; findings table with columns `Change | Classification | Evidence`.
3. **Migration notes** — one block per breaking change (Phase 6).
4. **Changelog fragment** — ready to paste into `CHANGELOG.md`, grouped by `### Added / ### Deprecated / ### Changed / ### Removed / ### Fixed` (Keep a Changelog format).

In `mode: changelog` emit only section 4. In `mode: gate` emit a single line: `apidiff: <N> breaking, <M> deprecation, <K> additive — <semver>`.

## What apidiff does not do

- Does not modify contracts. Read-only on both ends.
- Does not run the service. Static surface comparison only.
- Does not detect behavioural / semantic changes not reflected in the contract (e.g. a response field now returns milliseconds instead of seconds). Call those out as "undetectable by apidiff — human review required" where the heuristic suggests it.
- Does not enforce a version bump — it recommends.

## References (load on demand)

- `references/surface-detection.md`
- `references/openapi-rules.md`
- `references/grpc-rules.md`
- `references/nats-subject-rules.md`
- `references/go-public-rules.md`
- `references/ts-public-rules.md`
- `references/sql-public-rules.md`
