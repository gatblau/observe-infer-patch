# Logging hygiene

## What must never be logged

- Passwords (cleartext or hashed).
- API keys, session tokens, JWTs, refresh tokens, bearer tokens.
- Full request bodies of authentication or payment endpoints.
- Personal data whose regulation requires it not be logged (health, biometric, precise location) — depends on jurisdiction; flag for human review.
- Private keys, encryption keys, initialisation vectors.

## Patterns to flag

### Go
- `log.Printf("%+v", request)` where `request` contains auth fields.
- `logger.Info("body", "body", string(body))` with raw request body.
- Zap / Zerolog structured fields carrying `password`, `token`, `secret` as value.

### Python
- f-string log calls including `user`, `request`, `token`.
- `logger.debug(repr(obj))` on objects with secret fields.

### TS
- `console.log({ ...req })` dumping the request object.

## Severity

- **High:** log call includes a token/password/API key value, even at debug level (debug tends to end up enabled in prod accidentally).
- **Medium:** log includes full request/response bodies on auth routes.
- **Low:** identifier-only PII logged (user IDs, email addresses) without business justification.

## Safe patterns (not findings)

- Redacted logs: `logger.Info("auth", "user_id", id, "token", "[REDACTED]")`.
- Hashed identifiers for correlation.
- Structured logging with field allowlists.

## Evidence

Show the log statement and the source of each field that looks sensitive.
