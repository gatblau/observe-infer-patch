# Authentication and authorisation gaps

## HTTP route audit

List every route registered in the HTTP framework and check each for an auth middleware.

### Go / Echo
- `e.GET("/foo", handler)` without a middleware argument AND the route not inside an authenticated `Group` → unauthenticated.
- `e.Group("/api/v1", authMW)` covers all nested routes — note the pattern.

### Go / Gin, Chi
- `r.Use(authMW)` applies to all routes declared after it on the same router.
- Sub-router declared without inheriting the parent's middleware → unauthenticated.

### Express / FastAPI
- Route decorators lacking `@requires_auth` / `Depends(get_current_user)` — flag.

## Severity

- **Critical:** admin / write / destructive endpoint with no auth.
- **High:** read endpoint exposing sensitive data (user list, logs, tokens) with no auth.
- **Medium:** read endpoint exposing aggregated/anonymised data with no auth (review whether public).
- **Low:** health/ready endpoint without auth (usually intentional — verify).

## Authorisation (IDOR / privilege checks)

After confirming a handler has AuthN, check AuthZ:
- Does it verify the caller is permitted to act on the resource in the URL?
- Pattern: `GET /users/:id/data` handler that accepts any authenticated user without checking `id == caller.id` — IDOR.

Flag as **high** when a handler returns or mutates another tenant/user's data without an ownership check.

## Cross-tenant leakage

For multi-tenant codebases (this project is): every query against a tenant-scoped table must include `WHERE tenant_id = ?` from the caller's principal. Grep for queries on tenant tables that don't filter on tenant — flag each as **high**.

## Session / token handling

- JWT signed with `none` algorithm — **critical**.
- JWT with HMAC secret shorter than 32 bytes or in code — **high**.
- Session cookies without `HttpOnly`, `Secure`, `SameSite` attributes — **medium** each.
- Tokens compared via `==` rather than constant-time compare — **medium** (timing attack).

## Rate limiting on auth endpoints

Login / password-reset / MFA-verify endpoints without rate limiting — **medium**. Brute-force exposure.
