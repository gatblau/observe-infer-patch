# CI / CD supply-chain checks

## GitHub Actions (`.github/workflows/*.yml`)

- Third-party actions referenced by tag (`actions/checkout@v4`) rather than pinned SHA — **medium** (tag can be re-pointed).
- `pull_request_target` event combined with checkout of the PR ref — **critical** (code execution with write token).
- `permissions:` missing or set to full `write-all` at workflow level — **medium** (least-privilege violation).
- Use of `${{ github.event.* }}` values interpolated into `run:` script — **high** (script injection).
- Secrets referenced in workflows triggered by `pull_request` from forks — **high** (secret leakage).
- `workflow_dispatch` with inputs used unsanitised in shell — **medium**.

## GitLab CI (`.gitlab-ci.yml`)

- `rules:` missing on jobs that use protected variables — **medium**.
- Docker-in-docker (`docker:dind`) as a service on shared runners — **medium**.
- Scripts interpolating `$CI_COMMIT_TITLE` / user-controlled variables into shell — **high**.

## Generic

- Tokens with excessive scopes (`repo`, `admin:org`) used where narrower scopes would suffice — **medium**.
- No SBOM generation in release pipeline — **low**.
- No dependency-review step on PRs — **low**.
- Build artefacts uploaded without integrity hashes — **low**.
- Release signing absent (cosign, gpg) for distributed binaries — **medium** for public projects, **low** for internal.

## Branch / repo protections (observable via config)

Detect only what's in-repo:
- `CODEOWNERS` missing — **info**.
- `.github/dependabot.yml` / Renovate config missing — **low**.

Remote-only settings (required reviewers, signed commits required, status checks required) cannot be audited without GitHub/GitLab API — state that as a follow-up rather than flagging.
