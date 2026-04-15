# Weak cryptography

## Hashing

- MD5, SHA1 used for anything security-relevant (integrity, signing, password storage) — **high**. (Checksum-only use for non-adversarial integrity is **info**.)
- Password storage with plain SHA-256, SHA-512, or any non-KDF hash — **critical**.
- Password storage with bcrypt at cost < 10 — **medium**. Cost < 8 — **high**.
- `pbkdf2` with < 100 000 iterations — **medium**.
- Homebrew password schemes (`hash(salt + password)`) — **critical**.

## Symmetric encryption

- ECB mode — **high**.
- CBC without HMAC / authenticated mode — **medium** (padding oracle risk).
- Hardcoded IV or IV reuse — **high**.
- Key derived from a password without a proper KDF — **high**.
- Use of DES, 3DES, RC4 — **high**.

Prefer AES-GCM, ChaCha20-Poly1305, libsodium / NaCl / age.

## Asymmetric

- RSA key size < 2048 bits — **high**.
- RSA with PKCS#1 v1.5 padding on new code — **medium** (prefer OAEP/PSS).
- ECDSA with non-random `k` (deterministic variant fine per RFC 6979, but verify not hand-rolled) — **high** if hand-rolled.

## Randomness

- `math/rand` (Go), `Math.random` (JS), `random` (Python stdlib `random` module) used to generate tokens, session IDs, or crypto material — **high**.
- Correct: `crypto/rand`, `crypto.randomBytes`, `secrets` (Python).

## TLS config

- `InsecureSkipVerify: true` in `tls.Config` — **high** (verify not limited to test code).
- Minimum TLS version < 1.2 — **medium**; < 1.0 — **high**.
- Permitted cipher suites including RC4, export ciphers, NULL — **high**.

## Constant-time comparison

String comparison of MAC / token / signature values via `==` or `strings.Equal` — **medium**. Use `hmac.Equal` / `crypto.subtle.timingSafeEqual`.
