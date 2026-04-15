# Design document template

Every section is mandatory. Write "N/A — <reason>" for sections that genuinely do not apply. British English throughout.

```markdown
# <Feature / Problem Name>

## Summary
<Two to four sentences. What this design covers and why it exists. Non-goals noted briefly.>

## Motivation
<The current state, the pain or gap, and why doing nothing is not acceptable. Concrete — name users, systems, or incidents where possible.>

## Goals
- <Observable outcome 1>
- <Observable outcome 2>

## Non-goals
- <What this explicitly does not address, and why (brief).>

## Glossary
| Term | Definition | Example |
|---|---|---|
| <Domain noun> | <Definition> | <Concrete example value> |

## Domain Model
<For each entity: its purpose, key fields (without committing to data types yet — that is specgen's job), relationships to other entities, and lifecycle (created, updated, deleted, archived, etc.).>

## Actors and Triggers
<Who or what initiates each flow. User actions, scheduled jobs, external events, messages on a bus, API callers.>

## Flows
<For each significant flow: name it, list the steps in order, name the actor/trigger, and state the observable result. Use numbered lists. Mermaid sequence diagrams are welcome but optional.>

### Flow: <name>
1. <Step>
2. <Step>
- **Trigger:** <who/what starts this flow>
- **Result:** <observable outcome>
- **Error cases:** <what can go wrong, and the expected handling posture — retry, fail fast, alert, etc.>

## Data Lifecycle
<Where data originates, where it is stored, how long it is retained, who can read or modify it. Note any compliance, privacy, or multi-tenancy constraint.>

## Cross-cutting Concerns
<For each concern: state one of "applies — <brief policy>", "does not apply — <why>", or "project default — see <reference>". All must be stated explicitly; do not omit.>

| Concern | Stance |
|---|---|
| Authentication | |
| Authorisation | |
| Logging | |
| Metrics | |
| Tracing | |
| Rate limiting | |
| Pagination | |
| Input validation | |
| Error handling / envelope | |
| Configuration | |
| Health checks | |
| Migrations | |
| Graceful shutdown | |
| CORS | |
| Multi-tenancy | |

## Non-functional Requirements
- **Performance targets:** <latency, throughput, memory>
- **Scale:** <expected load, growth curve>
- **Availability:** <SLO, failure modes tolerated>
- **Security posture:** <authn, authz, secret handling, threat model summary>

## Interfaces (external)
<APIs, NATS subjects, CLI commands, file formats this feature exposes to the outside world. Names and shapes, not full schemas — that is specgen's job. Mark each as "new" or "extends existing".>

## Assumptions
<Every reasonable default chosen during interview. One row per assumption.>

| ID | Area | Assumption | Rationale |
|---|---|---|---|

## Open Questions
<Only items that materially affect architecture and are not yet decided. Mark each blocking / non-blocking.>

| ID | Question | Options | Impact | Blocking? |
|---|---|---|---|---|

## Risks
<Things that could derail the feature during build or after deployment. Each risk gets one line: what it is, and what we will do about it.>

## Alternatives considered
<At least one alternative approach with a one-line reason it was rejected. If genuinely none considered, state "None considered — <why>".>

## Dependencies
<Other features, teams, services, or infrastructure this design relies on.>

## Rollout and Rollback
<How the feature ships (dark launch, flag, full release) and how it is reverted if it goes wrong. For schema changes specifically, describe rollback semantics.>
```

## Rules for applying this template

- Write for a future reader who joins the team in six months. No in-jokes, no "as we discussed", no dangling references.
- Keep the Glossary, Domain Model, and Actions sections sharp — these are what `specgen` will lift directly into Phase 1C, Phase 2B/2C, and Phase 3.
- Do not invent technical constraints the user did not state. Defaults go in Assumptions, not hidden in prose.
- Cross-cutting Concerns must have all rows present. An empty row is a bug — it means the interview missed something.
