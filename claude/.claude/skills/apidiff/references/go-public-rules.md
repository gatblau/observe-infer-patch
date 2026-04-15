# Go public surface diff

Scope: all exported identifiers in non-`internal/` packages. `cmd/` binaries' exported identifiers are public only if another module imports them — in practice, out of scope.

## Additive

- New exported function, type, variable, constant, or method.
- New field added to a struct that is NOT constructed by external consumers via a composite literal (i.e. has an unexported field or lives behind a constructor).
- New method on an interface when the interface is NOT intended to be implemented by consumers (e.g. sealed by an unexported method).
- New implementation of an existing interface.

## Deprecation

- `// Deprecated: <reason>` comment added directly above the identifier.

## Breaking

### Identifier level
- Exported identifier removed or renamed.
- Package moved / renamed.
- Export-ness downgraded (exported → unexported).

### Function / method signature
- Parameter added, removed, reordered, or retyped.
- Return value added, removed, reordered, or retyped.
- Receiver changed from value to pointer or vice versa (subtle — flag and explain).
- Variadic changed to non-variadic or vice versa.

### Types
- Struct field removed or renamed.
- Struct field type narrowed.
- Struct that consumers construct via composite literal gaining a new required-to-initialise field (usually any new field before the first unexported field). Mitigate by adding an unexported field elsewhere — flag.
- Interface method added (breaks implementers) — unless sealed.
- Interface method removed (breaks consumers of that method).
- Type kind changed (struct → interface, named type → alias with different underlying type).

### Constants
- Untyped constant value changed (rare but breaking if consumers compare against it).
- Typed constant type narrowed.

### Generics
- Type parameter added to an existing function/type.
- Constraint narrowed on an existing type parameter.

## Evidence format

Use `go doc`-style output:

```
- Identifier: func (s *Server) RegisterUser(ctx context.Context, u *User) (string, error)
- Base: route/server.go:42
- Head: removed
- Classification: breaking
```

## Sealing pattern

Note when surfaces are sealed against consumer implementation. E.g. interfaces with an unexported method:

```go
type Database interface {
    Query(...) (...)
    sealed() // unexported — external packages cannot implement
}
```

On a sealed interface, adding a method is additive (no external implementer exists). Note the seal in the finding.
