# gRPC / protobuf diff classification

Use `buf breaking` where available — it encodes the Google API style guide. This reference documents the same rules for when buf is not set up.

## Additive

- New service.
- New RPC method on an existing service (caller opts in).
- New message.
- New field in a message with a new, unused field number.
- New enum value appended.
- New option / annotation that doesn't change wire compatibility.

## Deprecation

- `option deprecated = true;` on a method, message, field, or enum value.
- `[deprecated = true]` inline on a field.

## Breaking

### Services / methods
- Service removed or renamed.
- RPC method removed or renamed.
- RPC method's request or response message changed to a different type.
- Streaming behaviour change (unary → streaming, or any stream-direction change).

### Messages / fields
- Field removed (wire-breaking if clients populate / read it).
- Field number changed.
- Field number reused for a different semantic field.
- Field name changed (JSON-mapped breaking; wire compat preserved).
- Field type changed (except for wire-compatible pairs: `int32 ⇄ int64` is narrowing/widening risk — flag).
- Field cardinality changed (`optional` ⇄ `repeated`, `required` removal legal only in proto3).
- Oneof membership changed (field moved into/out of a oneof).
- `map<K,V>` type parameters changed.

### Enums
- Enum value removed.
- Enum value's number changed.
- Enum first value (position 0) changed — sentinel shift.

### Packages
- Package renamed.
- File moved to a different package.
- `go_package` option changed in a way that breaks generated import paths.

## Wire-compat vs source-compat

Note both when they differ:
- Field rename: wire-compatible (field number unchanged), source-breaking for JSON consumers and generated bindings.
- Type `int32 → int64`: wire-compatible on the happy path but values > INT32_MAX silently truncate for old clients — flag as breaking.

## Evidence format

```
- Service: user.v1.UserService
- RPC: GetUser
- Change: request field `id` type changed from `string` to `bytes`
- Base: .proto L42
- Head: .proto L42
```
