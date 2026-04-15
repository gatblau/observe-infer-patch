# Secrets and keys scan

Scan for credentials committed to the repo. Always run; never skip regardless of scope.

## High-signal patterns (regex)

Run each pattern across all text files (exclude `.git/`, `node_modules/`, `vendor/`, `*.lock`, binary files).

| Token type | Pattern |
|---|---|
| AWS access key | `AKIA[0-9A-Z]{16}` |
| AWS secret key (context) | `aws_secret_access_key\s*=\s*['"][A-Za-z0-9/+=]{40}['"]` |
| GitHub PAT (classic) | `ghp_[A-Za-z0-9]{36}` |
| GitHub fine-grained | `github_pat_[A-Za-z0-9_]{82}` |
| GitLab PAT | `glpat-[A-Za-z0-9_-]{20}` |
| Slack token | `xox[baprs]-[A-Za-z0-9-]{10,}` |
| Google API key | `AIza[0-9A-Za-z_-]{35}` |
| Stripe live key | `sk_live_[0-9a-zA-Z]{24,}` |
| Private key block | `-----BEGIN (RSA |EC |OPENSSH |PGP )?PRIVATE KEY-----` |
| JWT | `eyJ[A-Za-z0-9_-]{10,}\.eyJ[A-Za-z0-9_-]{10,}\.[A-Za-z0-9_-]{10,}` |
| Generic high-entropy | 32+ char Base64/hex assigned to `*secret*`/`*token*`/`*password*` identifier |

## Low-signal indicators (require context)

- Identifiers matching `(?i)(password|passwd|pwd|secret|api_?key|access_?key|private_?key|token)` with a string literal value.
- `.env`, `.env.*` files not in `.gitignore`.
- Hardcoded database URLs with inline credentials: `postgres://user:pass@host`.

## Report format

Never include the secret value in the report:
```
F-### — Possible {{token_type}} committed  (high|critical)
Location: path/to/file:line
Evidence: (redacted — matched pattern for {{token_type}})
```

## Triage

- **Critical:** matches a live-format token regex AND the file is tracked in git (not a test fixture marked obviously fake).
- **High:** matches a live-format token in an ignored-but-present `.env`-style file (risk of future commit).
- **Medium:** identifier + literal where value looks like a placeholder (`changeme`, `xxx`, 32-char hex).
- **Info:** test fixtures with obviously fake credentials (`test`, `example`, `AAAA...`).

Always recommend rotation for any `critical` finding, regardless of whether git history also contains it — assume compromise.

## Git history

State as a follow-up: "If a secret is confirmed, rotate it and consider `git filter-repo` / BFG to purge from history. Purging is destructive; coordinate with the team before force-pushing."
