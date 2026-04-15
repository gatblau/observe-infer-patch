# Transport, CORS, headers

## CORS

- `AllowOrigin: "*"` combined with `AllowCredentials: true` — **high** (browser blocks it, but intent is dangerous).
- `AllowOrigin: "*"` on any endpoint that reads authenticated state — **medium**.
- Echo / Gin `cors.Default()` with credentials — verify.
- Reflecting `Origin` header without an allowlist — **high**.

## Security headers

Missing or weak values — each **low** unless combined with other risk:
- `Strict-Transport-Security` — recommended `max-age=31536000; includeSubDomains`.
- `Content-Security-Policy` — missing on HTML responses.
- `X-Content-Type-Options: nosniff`.
- `X-Frame-Options: DENY` or `frame-ancestors` in CSP.
- `Referrer-Policy: strict-origin-when-cross-origin`.

## HTTP vs HTTPS

- Server listens on HTTP without an upstream TLS terminator — **medium**. Often intentional behind a load balancer; note and verify.
- Mixed HTTP/HTTPS redirects that leak auth cookies — **high**.

## Cookies

- Session / auth cookie set without `Secure` flag — **medium**.
- Without `HttpOnly` — **medium**.
- Without `SameSite` — **medium** (CSRF exposure depending on framework).

## Open redirects

Any handler that reads a `redirect`, `next`, `url` query parameter and returns an HTTP 30x to it without validating against an allowlist — **medium**.

## SSRF

Server-side HTTP calls built from user-supplied URLs without an allowlist — **high**. Typical sinks: `http.Get(userURL)`, `requests.get(user_url)`, image/avatar fetchers, webhook senders.
