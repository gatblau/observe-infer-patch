# Alert mapping

How to turn concrete failure signals in code into "Alert responses" entries an operator can act on under pressure.

## Entry shape (repeat per signal)

```markdown
### Alert: <short symptom>

- **What you see:** <log string / metric name / HTTP status>
- **First look:** <one command or file to check>
- **Likely cause:** <one-line hypothesis, cited>
- **Escalate when:** <condition that makes this an incident>
```

Keep each to four bullets. If you need more, split into two alerts.

## Deriving each field

### What you see

Prefer the exact string the operator will actually encounter:

- A log message → use the format string as-authored.
- A metric → use the metric name and a threshold if the code defines one.
- An HTTP status → state the route that returns it, not just the code.
- A user-visible symptom (e.g. "requests hang") only when no internal signal exists.

### First look

One command. It must be something the operator can run without extra context. Examples that work:

- `kubectl logs -l app=<service> --tail=200 | grep '<substring>'`
- `curl -s localhost:<port>/healthz`
- `psql -c "select count(*) from <table> where <column> > now() - interval '5 min'"` (only if `<table>` is verified to exist)

Never invent commands that depend on unverified topology (e.g. `kubectl` in a docker-compose environment).

### Likely cause

One sentence, citing the line(s) that emit the signal. If there are several plausible causes, pick the most common one for the cited pattern and list the others under the service's "Known failure modes" section, not here.

### Escalate when

The condition that turns triage into an incident. Examples:

- Signal persists beyond one shutdown/startup cycle.
- Error rate exceeds `<n>` per minute as per the Prometheus counter in `path:line`.
- Dependency `<X>` is confirmed healthy but the signal continues.

Do not write "if it doesn't resolve" without a specific observable condition.

## Prioritisation

Cap alert responses at 5–8 entries. Select by:

1. **Panic / fatal log** anywhere on the request path — always include.
2. **Sentinel errors** returned from a public handler — include if they map to a user-visible failure.
3. **Generic errors** (wrapped `fmt.Errorf` with dynamic content) — include only if a Prometheus counter or log severity elevates them.
4. **Debug-level logs** — exclude.

State the omit count at the end of the section: "Omitted N lower-severity signals — see source for details."

## What to skip

- Signals that only fire during tests or in `_test.go` / `__tests__/` files.
- Log lines in scripts not part of the runtime binary.
- Errors that are caught and retried internally with no external effect — they are implementation detail, not alerts.

## Correlation with dependencies

If a signal is clearly a symptom of a dependency failure (DB connection refused, NATS disconnect, upstream 5xx), cross-reference the dependency's row in the "Dependencies" table. The alert response should point the operator at the dependency first, not at the service itself.
