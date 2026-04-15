# OpenAPI diff classification

## Additive

- New path.
- New HTTP method on an existing path.
- New optional request parameter (query, header, body field).
- New response field.
- New enum value on a response (tolerant-reader assumption — see Breaking notes below).
- New `200`-series response code.
- New tag, new description, new example.

## Deprecation

- `deprecated: true` added to an endpoint, parameter, or schema property.
- Description updated to include "deprecated" / "use X instead".

Deprecation is never itself breaking, but schedule the removal.

## Breaking

### Path / method
- Path removed.
- Method removed on an existing path.
- Path renamed.

### Request
- Required parameter added.
- Optional parameter made required.
- Parameter renamed (query/header/body).
- Parameter type narrowed (`integer` → `integer` with tighter `maximum`, `string` → `string` with pattern).
- Parameter location changed (query → header).

### Response
- Response field removed.
- Response field renamed.
- Response field type changed in a non-widening way.
- `200`-series response code removed.
- New `4xx`/`5xx` response that previously succeeded for a valid request.

### Schema
- Schema removed.
- `required` extended with a previously optional property (in a request schema).
- `additionalProperties` changed from `true` to `false`.
- `oneOf` / `anyOf` member removed.

### Security
- Security scheme added to an endpoint that was previously open.
- Scope requirements broadened.

## Ambiguous — flag for user

- **New enum value in response:** breaking for strict consumers, additive for tolerant ones. Default to breaking if the API's style guide requires strict; otherwise surface the choice.
- **Response field made required:** technically a tightening but consumers already relied on it. Usually additive in practice — flag.
- **Description-only change that implies semantic change:** e.g. "amount in cents" → "amount in minor units". Cannot be classified from the schema alone. Flag as "undetectable semantic change — human review".

## Evidence format

For each finding:
```
- Path: POST /users/{id}/avatar
- Change: request field `image_url` made required
- Base: optional
- Head: required
- Location: paths./users/{id}/avatar.post.requestBody.content.application/json.schema.required
```
