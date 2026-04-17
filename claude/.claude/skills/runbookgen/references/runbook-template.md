# Runbook template

Markdown skeleton for runbookgen output. Keep sections in this order. Omit a section entirely rather than padding it with "N/A" — an empty section is worse than no section.

Use British English in narrative prose; keep identifiers and commands as-authored.

---

```markdown
# <Service name> — Runbook

**Repo:** <repo name>
**Entry point:** <path/to/main.go>:<line>
**Audience:** <on-call | new-joiner>
**Generated:** <date>

## Service summary

One paragraph. What the service does, who calls it, what breaks if it stops. Anchor the "who calls it" claim to code if possible (e.g. "consumed by X over NATS subject Y — see path:line"), otherwise mark it as **Needs operator confirmation**.

## At a glance

| Field | Value |
|---|---|
| Binary / container name | `<name>` |
| Image or build artefact | `<image:tag>` or `<artefact path>` |
| Primary ports | `<port/protocol, ...>` |
| Primary config path | `<path>` |
| Entry point | `<path:line>` |
| Deployment manifest | `<path>` or `Needs operator confirmation` |

## Start / stop

**Start a single instance:**
```
<exact command or manifest reference>
```

**Stop gracefully:**
```
<exact command — signal, compose down, kubectl delete, etc.>
```

State the graceful-shutdown timeout if the code sets one, with citation.

## Configuration

| Name | Default | Source | Secret? | Notes |
|---|---|---|---|---|
| `<ENV_VAR>` | `<default or none>` | `path:line` | yes/no | short explanation |

Only list variables actually read by the code. Do not list variables that only appear in sample files unless the code reads them.

## Dependencies

| Dependency | Role | Failure impact | Source |
|---|---|---|---|
| `<name>` | `<db / queue / cache / upstream http>` | `<what breaks>` | `path:line` |

## Health checks

List endpoints or their absence. Example:

- `GET /healthz` → 200 when the process is alive. Source: `path:line`.
- `GET /readyz` → 200 when dependencies are reachable. Source: `path:line`.
- **No readiness endpoint detected.** Needs operator confirmation.

## Alert responses

One subsection per signal. Keep each short — on-call reads these under pressure.

### Alert: <short name of symptom>

- **What you see:** log string / metric / HTTP status the operator observes.
- **First look:** one command or file to check.
- **Likely cause:** one-line hypothesis, with citation if derivable from code.
- **Escalate when:** the condition that turns triage into an incident.

## Known failure modes

Short bullets. Each cites code.

- `<short description>` — `path:line`.

## Rollback

Exact steps. If multiple rollback paths exist (image tag, migration, feature flag), list them in order of least to most disruptive.

```
<command>
```

If rollback is not detectable from the repo, write: **Rollback path not detectable from the repo — document manually.**

## Gaps

Anything marked "Needs operator confirmation" during drafting. A human fills these in before the runbook is considered complete.

- `<gap description>`

## Escalation

(Full scope only.) Placeholders unless the repo states contacts.

- **Primary on-call:** `<team / rotation>`
- **Secondary:** `<team>`
- **Paging channel:** `<channel / system>`
```
