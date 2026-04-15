# Dependency audit

Static inspection of dependency manifests. Does not fetch vulnerability databases — surfaces the command for the user to run themselves.

## Per-ecosystem commands

| Ecosystem | Command |
|---|---|
| Go | `govulncheck ./...` (install: `go install golang.org/x/vuln/cmd/govulncheck@latest`) |
| Node.js | `npm audit` or `pnpm audit` or `yarn audit` |
| Python | `pip-audit` (install: `pip install pip-audit`) |
| Rust | `cargo audit` (install: `cargo install cargo-audit`) |
| Java (Maven) | `mvn dependency-check:check` (OWASP plugin) |
| Generic SBOM | `syft <repo> -o spdx-json` then `grype sbom:<file>` |

Include these as the **next step** in the report rather than attempting to run them inside the skill. Running them may download content and is outside the scope of static inspection.

## Static checks (no network)

- Direct dependencies pinned vs floating ranges. `^1.2.3` / `~1.2.3` / `latest` on security-critical packages is **low** (operational risk, not vulnerability).
- Lockfile missing (`package-lock.json`, `yarn.lock`, `go.sum`, `Cargo.lock`) — **medium**. Non-reproducible builds.
- Git-URL dependencies (`github:user/repo` without a commit SHA) — **medium**. Supply-chain risk.
- Dependencies from non-canonical registries — **medium**. Review whether typo-squatting.
- Abandoned / archived upstreams (flag any dependency whose repo is visibly archived — may require WebFetch, keep off by default).

## Report format

```
F-### — Unpinned dependency `foo` in package.json  (low)
Suggested: pin to a specific version or commit.
Next: run `npm audit` for the current vulnerability picture.
```

One consolidated finding per manifest is fine; do not generate one finding per dependency unless each has an independent risk.
