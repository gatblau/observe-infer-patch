---
name: secscan
description: Audit the repository for security posture issues — hardcoded secrets, insecure defaults, vulnerable dependencies, missing authentication/authorization, unsafe SQL/shell construction, weak crypto choices, permissive CORS, and logging of sensitive data. Produces a ranked findings report. Trigger on phrases like "security review", "secscan", "audit for secrets", "check for vulnerabilities", "security posture", "secure code review", or when the user asks for a defensive scan of the codebase. Do NOT trigger for offensive security work, exploit development, or penetration tests against systems you do not own.
---

# secscan — Defensive security posture audit

Produce a ranked list of security posture issues discovered by static inspection of the repository. Code-only review: no network scans, no runtime probing, no credential testing.

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Arguments

- `scope`: default the whole repo. Accepts a path (`lib/`), a package (`./route`), or a concern (`secrets`, `sql`, `auth`, `crypto`, `deps`, `logging`, `transport`).
- `mode`: `report` (default — findings only) | `fix-proposals` (add suggested remediations per finding, still no code writes) | `triage` (findings + prioritisation for an issue backlog).
- `severity-floor`: default `low`. Suppresses findings below this threshold. One of `info`, `low`, `medium`, `high`, `critical`.

## Active mode

**Observation + Inference.** Observation for evidence gathering (Phase 1–2: file paths, line numbers, code patterns). Inference for severity assignment (Phase 3: threat-model reasoning — "what could an attacker do, given these pre-conditions"). Never transitions to Patch — even `fix-proposals` mode emits text, not edits. Findings are evidence-tagged; severity claims are hypothesis-tagged with stated pre-conditions.

## Hard rules

- **Static inspection only.** No execution of untrusted input, no network calls, no credential verification.
- **Never exfiltrate findings.** The report stays in the conversation. Do not write it to pastebins, gists, or third-party services.
- **Severity must be justified.** Every finding states (a) what the risk is, (b) what an attacker could do, (c) the pre-conditions for exploitation. A finding without a concrete threat model is rewritten as an "observation" at `info` severity.
- **Do not fix.** Even in `fix-proposals` mode, surface the fix as text. Actual edits require a follow-up `/apply` invocation or explicit user approval.
- **Respect confidentiality.** If a finding contains what appears to be a live secret (AWS key format, GitHub token format, a .pem), state only the file path and line number, never the secret value in the report body.

## Process

### Phase 1 — Scope and language probe
Determine:
- Languages present (Go, TS, Python, SQL, shell, Dockerfile, YAML, Terraform — each has different checks).
- Framework signals (Echo, Gin, Express, FastAPI, Spring — each has framework-specific checks).
- Dependency manifests (`go.mod`, `package.json`, `requirements.txt`, `Cargo.toml`, `pom.xml`).
- CI/CD files (`.github/workflows`, `.gitlab-ci.yml`).

Reuse the extraction tiers defined in `.claude/skills/syncheck/references/extraction-tiers.md` (T1 filesystem → T4 AST) for language-specific parsing. Do not duplicate that logic here.

### Phase 2 — Concern-driven scan

Run each concern the scope implies. Load the relevant reference on demand.

| Concern | Reference | Triggers |
|---|---|---|
| Secrets & keys in code | `references/secrets-patterns.md` | Always. |
| SQL injection | `references/sql-injection.md` | `**/*.go`, `**/*.py`, `**/*.ts` with string concatenation near SQL. |
| Shell / command injection | `references/command-injection.md` | `exec.Command`, `os.system`, `subprocess`, `child_process`. |
| AuthN/AuthZ gaps | `references/auth-gaps.md` | HTTP handlers, middleware chains, route registrations. |
| Weak crypto | `references/weak-crypto.md` | `crypto/*`, `hashlib`, `bcrypt`, hardcoded IVs, MD5/SHA1 usage. |
| Insecure transport / CORS | `references/transport.md` | HTTP servers, CORS middleware, TLS config. |
| Vulnerable dependencies | `references/dep-audit.md` | Presence of a dependency manifest. |
| Sensitive data in logs | `references/logging-hygiene.md` | Logger calls with variable interpolation near auth/user data. |
| Container / deployment | `references/container-hardening.md` | `Dockerfile`, `docker-compose*.yml`, k8s manifests. |
| CI/CD supply chain | `references/ci-supply-chain.md` | `.github/workflows/**`, `.gitlab-ci.yml`, pipeline files. |

### Phase 3 — Severity assignment

For each finding, assign severity from the matrix:

| Severity | Criteria |
|---|---|
| Critical | Remote unauth exploit, leaked live secret, broken auth on admin surface, RCE primitive. |
| High | Remote authenticated exploit leading to privilege escalation, SQLi on non-admin route, secret committed but rotated/dev-only. |
| Medium | Logic flaw requiring specific conditions, weak crypto on non-critical path, missing rate limit on auth. |
| Low | Defence-in-depth gap (e.g. missing security header) with no direct exploit path. |
| Info | Observation with no demonstrable threat. |

Record the reasoning per finding. A finding with no stated threat model is not a finding — downgrade or drop.

### Phase 4 — Report assembly

Emit the report in this shape:

```markdown
# secscan report

**Scope:** {{scope}}
**Severity floor:** {{floor}}
**Generated:** {{UTC timestamp}}

## Summary
{{counts by severity}}

## Findings

### F-001 — {{title}}  ({{severity}})
- **Location:** `path/to/file.go:42`
- **Concern:** {{concern category}}
- **What an attacker could do:** {{concrete impact}}
- **Pre-conditions:** {{what access / inputs are needed}}
- **Evidence:**
  ```
  {{quoted code, minimal}}
  ```
- **Suggested remediation (mode=fix-proposals only):** {{short fix}}

### F-002 — ...
```

Sort findings descending by severity, then by file path.

### Phase 5 — What to do next

End with three concrete next steps tied to the findings:

1. Top critical/high fixes — exact file paths and which skill/command to invoke (`/apply`, `schemagen`, etc.).
2. Dependency updates with a single `go get -u ./...` / `npm audit fix` style command if applicable.
3. Gaps requiring human judgement (e.g. "review whether `/api/v1/admin/*` should accept API-key auth or only session auth").

## What secscan does not do

- Does not execute the code.
- Does not perform dynamic testing, fuzzing, or network probes.
- Does not check secrets against issuer APIs (would constitute exfiltration).
- Does not fix issues directly — proposals only.
- Does not audit external systems referenced by the code (e.g. the configuration of a cloud IAM role); it flags the fact that such a review is needed.

## References (load on demand)

- `references/secrets-patterns.md`
- `references/sql-injection.md`
- `references/command-injection.md`
- `references/auth-gaps.md`
- `references/weak-crypto.md`
- `references/transport.md`
- `references/dep-audit.md`
- `references/logging-hygiene.md`
- `references/container-hardening.md`
- `references/ci-supply-chain.md`
