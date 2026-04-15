# TypeScript public surface diff

Scope: identifiers exported from entries listed in `package.json` `"exports"` / `"main"` / `"types"`, and transitively re-exported from there.

## Additive

- New exported function, class, type, interface, enum, const.
- New optional property on an exported interface / type.
- New overload of an exported function.
- New method on a class, if the class is not typically subclassed.
- New optional parameter at the end of a function signature.

## Deprecation

- `@deprecated` JSDoc tag on an exported symbol.

## Breaking

### Identifier level
- Exported identifier removed or renamed.
- Default export changed to named or vice versa.
- File path / entry-point change that drops a re-export.

### Function signature
- Parameter added (unless optional and at the end).
- Parameter type narrowed.
- Parameter made required.
- Return type narrowed.
- Overload removed.

### Types / interfaces
- Required property added to an interface (breaks implementers).
- Property removed or renamed.
- Property type narrowed.
- Index signature removed.
- Generic type parameter added, removed, or reordered.
- Generic constraint narrowed.

### Enums
- Enum member removed or renamed.
- String-literal-union narrowed.

### Classes
- Constructor signature changed.
- Public method removed.
- Protected member removed (breaks subclasses).
- Class turned `abstract` or constructor made private.

### Module / package
- `exports` subpath removed.
- `types` field pointing at a narrower declaration file.
- `sideEffects` widened from `false` to `true` (tree-shaking breakage).

## Structural typing notes

TypeScript's structural typing means adding a required property to a type used as a function parameter is breaking for all callers, even if they pass an object literal that now lacks the property. Flag with this framing.

## Evidence format

```
- Export: function processRequest(req: Request): Promise<Response>
- Base: src/index.ts:15
- Head: src/index.ts:15, new required parameter `opts: Options`
- Classification: breaking
```
