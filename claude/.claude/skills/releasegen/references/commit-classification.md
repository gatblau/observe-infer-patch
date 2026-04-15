# Commit classification

Classify each commit in the range into one of the types below. Order: try conventional first, heuristic second, fallback "other".

## Conventional Commits

Format: `<type>(<scope>)?!?: <subject>` with an optional body. Types recognised:

| Type | Bucket |
|---|---|
| `feat` | Added |
| `fix` | Fixed |
| `perf` | Changed |
| `refactor` | Changed |
| `revert` | Reverted (usually omit, see SKILL) |
| `docs` | Docs (filter for external audience) |
| `test` | Tests (filter) |
| `build` | Build (filter) |
| `ci` | CI (filter) |
| `chore` | Chore (filter) |
| `style` | Style (filter) |

Breaking markers:
- `feat!:` or `<type>(scope)!:` — breaking.
- Body contains `BREAKING CHANGE:` or `BREAKING-CHANGE:` on its own line — breaking.

## Heuristic fallbacks

Apply only when the subject does not match a conventional prefix.

| Signal in subject (case-insensitive) | Bucket |
|---|---|
| `add`, `implement`, `introduce`, `support for` | Added (feat-equivalent) |
| `fix`, `resolve`, `correct`, `repair`, `patch` | Fixed |
| `remove`, `drop`, `delete` | Removed |
| `deprecate` | Deprecated |
| `refactor`, `clean up`, `simplify`, `rework` | Changed |
| `improve performance`, `optimise`, `speed up` | Changed (performance) |
| `security`, `CVE`, `vulnerability` | Security |
| `bump`, `update <dep>`, `upgrade <dep>` | Chore (filter) |
| `typo`, `doc`, `readme`, `comment` | Docs (filter) |
| `test`, `spec`, `coverage` | Tests (filter) |

## Breaking-change heuristics when no conventional marker

Apply sparingly — false positives are worse than false negatives:
- Subject contains `BREAKING`, `breaking change`, `remove public`, `remove exported`.
- Subject mentions removal of a route (`/api/` path) or env var.
- A commit touches `schema/v1/migrations/` with a down-migration noted as lossy.

Still flag as breaking only after reading the commit body. If unclear, classify as "other — manual review needed" and surface it.

## Recording per commit

For each commit produce:
```
{
  sha: "abc1234",
  type: "feat",
  scope: "auth",
  subject: "add NATS JWT rotation",
  breaking: false,
  author: "name <email>",
  bucket: "Added"
}
```

"other" commits are reported in a dedicated "Needs manual classification" section, not dropped silently.
