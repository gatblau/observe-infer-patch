---
name: runbookgen
description: Draft an operational runbook for a service or component from its source code and configuration. Covers what the service does, how it starts and stops, what it depends on, what to do when its known failure modes fire, and how to roll back a bad deployment. Produces Markdown suitable for a wiki page or docs/runbooks/ entry. Trigger on phrases like "runbookgen", "generate a runbook", "operational runbook for X", "SRE doc for X", "on-call guide for X", or when the user asks to document how to run/operate a service. Do NOT trigger to actually deploy, page, or run ops commands — this skill only drafts text.
---

# runbookgen — Operational runbook draft

Turn a running service's source code and configuration into a runbook an on-call engineer can use: overview, startup/shutdown, dependencies, alert responses, rollback, known failure modes. Drafting only — no deploys, no pages, no mutations.

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Arguments

- `target`: the service/component to document. Path, package name, container name, or binary name. Required.
- `scope`: `minimal` (overview + startup/shutdown + rollback) | `standard` (default — adds dependencies + alert responses + known failure modes) | `full` (adds capacity notes + escalation section).
- `output`: `inline` (default for small runbooks) | `file` (writes `docs/runbooks/<target>.md`).
- `audience`: `on-call` (default — terse, action-oriented) | `new-joiner` (adds short explanations of why each step exists).

## Active mode

**Observation.** Drafts text only — no `docker`/`kubectl` commands executed, no deploys, no edits to source or config. Suggested diagnostic commands are emitted as fenced blocks for the user to run themselves.

## Hard rules

- **Every claim traces to source.** Every env var, port, endpoint, config key, signal, dependency, log string, and health check must cite `path:line` in the repo. Anything not found is marked **Needs operator confirmation**, never guessed.
- **No invented endpoints.** If the runbook says `GET /healthz` returns 200, that route must exist in the code.
- **No secrets in output.** Example values for env vars use placeholders (`<REDACTED>`, `<your-token>`) even if the repo checks in a sample.
- **No destructive suggestions without rollback.** Every "restart / redeploy / migrate" step includes a rollback one-liner or explicit note that rollback is manual.
- **British English** for narrative sections. Command names and code identifiers stay as-authored.

## Process

### Phase 1 — Resolve target
- Confirm `target` resolves to exactly one runnable unit: a `main` package, a Dockerfile, a systemd unit, a Kubernetes manifest, or a named binary in the build config.
- If ambiguous (multiple `main.go`, multiple containers), ask the user which one. Do not pick silently.
- Capture: repo-relative path of the entry point, build/deploy artefact name, primary config file(s).

### Phase 2 — Extract operational facts
Load `references/extraction-signals.md`. For the target, gather with `path:line` citations:

- **Startup:** entry point, init order, flags parsed, env vars read, config files loaded.
- **Shutdown:** signal handlers (`SIGTERM`/`SIGINT`), graceful shutdown timeouts, context cancellation chains.
- **Ports & endpoints:** every listener (HTTP, gRPC, metrics, NATS subject subscriptions) with the line that binds or subscribes.
- **Dependencies:** outbound calls (DB DSN source, NATS servers, HTTP clients, object stores) and required infra.
- **Health & liveness:** `/healthz`, `/readyz`, metrics endpoints, or their absence.
- **Persistent state:** volumes, migration files under `schema/**`, any on-disk caches.

Anything expected but not found goes to the runbook's **Gaps** section rather than being fabricated.

### Phase 3 — Derive alert responses (standard/full only)
Load `references/alert-mapping.md`. For each concrete failure signal in the code — `log.Error` lines, `panic`, sentinel error strings, returned HTTP 5xx paths, circuit-breaker trips, metric thresholds if a `prometheus` counter exists — produce a triage entry: what the operator sees, where to look first, one diagnostic command (as text, not executed), and when to escalate.

Cap at the top 5–8 signals for readability. Note how many were omitted.

### Phase 4 — Draft rollback
Look at: deployment manifests (`k8s/**`, `docker-compose*.yml`, `helm/**`), migration directories, previous releases' tags, and any version pinning. Draft a rollback one-liner for:

- Re-deploy previous image/tag.
- Revert a migration if the migration is reversible (cite the `down` script); flag if not.
- Feature-flag toggle if one exists for this service.

If none of these are detectable, state: "Rollback path not detectable from the repo — document manually."

### Phase 5 — Known failure modes (standard/full only)
Scan for patterns that commonly bite operators: unbounded retries, goroutine leaks on cancelled contexts, missing timeouts on HTTP clients, cascading failures when a dependency is down. Only include patterns you can cite in code. Skip the section if none apply — do not pad.

### Phase 6 — Assemble output
Load `references/runbook-template.md`. Produce the Markdown sections in order:

1. **Service summary** — one paragraph, what it does and who depends on it.
2. **At a glance** — table of binary name, image, ports, primary config path, repo entry point.
3. **Start / stop** — exact commands or manifests that start and stop a single instance.
4. **Configuration** — every env var / flag with its default, source citation, and whether a secret.
5. **Dependencies** — table of each external dependency, its role, and failure impact.
6. **Health checks** — endpoints or their absence.
7. **Alert responses** (standard/full) — one subsection per signal.
8. **Known failure modes** (standard/full) — short bullets.
9. **Rollback** — the exact steps.
10. **Gaps** — everything marked "Needs operator confirmation."
11. **Escalation** (full only) — who to contact / where to page. Left as placeholders unless the repo states it.

### Phase 7 — Present
If `output: file`, write to `docs/runbooks/<target>.md` — do not overwrite without confirming. Otherwise return inline. Always reply with: the target resolved, the scope used, the section list, and the Gaps count so the user knows what still needs human input.

## Output files

- `output: file` → `docs/runbooks/<target>.md` (create `docs/runbooks/` if it does not exist).
- `output: inline` (default for `minimal` scope) → in-chat Markdown only.

## References (load on demand)

- `references/runbook-template.md`
- `references/extraction-signals.md`
- `references/alert-mapping.md`

## What runbookgen does not do

- Does not deploy, restart, page, or execute any diagnostic command. All suggested commands are text the user runs themselves.
- Does not modify source code or configuration. If the runbook-drafting process surfaces a bug (e.g. missing signal handler), the fix goes through `/rca` → `/breakdown` → `/apply`.
- Does not invent escalation contacts, SLAs, or on-call rotations. Those are placeholders unless the repo states them.
- Does not audit security posture (that is `secscan`), check API contract stability (that is `apidiff`), or critique SQL (that is `dbaudit`).
