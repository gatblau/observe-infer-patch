# Surface detection

For each surface type, the detection signal and the extraction strategy.

## OpenAPI / Swagger

**Signal files:**
- `openapi.yaml` / `openapi.json` at repo root or under `docs/`, `api/`.
- `swagger.yaml` / `swagger.json`.
- `docs/docs.go` + `docs/swagger.*` — swaggo-generated in Go projects.

**Extraction:** parse YAML/JSON to a spec tree. Key sections: `paths.*`, `components.schemas.*`, `components.parameters.*`, `components.responses.*`, `security.*`.

## gRPC / protobuf

**Signal files:**
- `*.proto` under `proto/`, `api/`, `pkg/proto/`, `rpc/`.
- `buf.yaml` indicates a buf workspace.

**Extraction:** prefer `buf` if present (`buf breaking --against <base>`), otherwise parse `.proto` files. Key elements: services, RPC methods, request/response messages, field numbers, enum values, options.

## NATS subjects

**Signal:**
- Go: calls to `nats.Subscribe`, `nats.QueueSubscribe`, `nats.Publish`, `js.Subscribe`, `js.Publish`.
- Subject string literals of shape `foo.*.bar.*.baz`.

**Extraction:** use `.claude/skills/syncheck/references/extraction-tiers.md` (T2 regex is usually sufficient for subjects). Record:
- Subject pattern.
- Publisher vs subscriber.
- Payload type (often a proto or JSON struct — name it if discoverable).
- Queue group.

## Go public surface

**Scope:** all exported identifiers in packages outside `internal/`, `cmd/`, `tools/`.

**Extraction:** use `go doc -all <pkg>` or parse the package with `go/parser`. Record for each exported symbol:
- Kind (func, type, var, const, method).
- Signature (for funcs/methods).
- Struct fields (for types).
- Interface method set.

## TypeScript public surface

**Scope:**
- Entries in `package.json` `"exports"` field (modern).
- `"main"` / `"types"` (legacy).
- `index.ts` re-exports.

**Extraction:** `tsc --emitDeclarationOnly` to produce `.d.ts`, then diff those. Lighter alternative: regex over `export` statements at the entry points.

## SQL public

**Scope:** `schema/v1/views/` and `schema/v1/funcs/` — anything callable from outside the DB.

**Extraction:** parse each `CREATE VIEW` / `CREATE FUNCTION` statement. Record:
- View/function name.
- Column list and types (views) / parameter list and return type (functions).
- SECURITY DEFINER / INVOKER mode.

Tables are deliberately NOT a public surface unless they are in `schema/v1/views/` — table changes ripple via schemagen, not apidiff.

## Surface presence matrix

Produce at the start of the diff:

```
| Surface        | Base | Head |
|----------------|------|------|
| OpenAPI        | yes  | yes  |
| gRPC           | no   | no   |
| NATS           | yes  | yes  |
| Go public      | yes  | yes  |
| TS public      | no   | no   |
| SQL public     | yes  | yes  |
```

Only diff the surfaces where both columns say yes OR where a presence change itself is the finding.
