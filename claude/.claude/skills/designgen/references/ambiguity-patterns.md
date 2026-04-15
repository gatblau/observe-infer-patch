# Ambiguity patterns

Re-read the gathered answers. For each pattern matched below, decide per the rule. This step most directly affects downstream spec quality.

## Language patterns that signal ambiguity

| Phrase in interview answer | Likely meaning | Action |
|---|---|---|
| "fast", "quickly", "snappy" | No latency target set | Record as Assumption with a concrete figure (e.g. <200ms p95) unless the user objects |
| "secure" | Security posture undefined | Assumption: authn scheme, authz model, transport security, at-rest encryption — each as a separate row |
| "scalable" | Scale target undefined | Ask once: rows/sec, messages/sec, peak vs average. If still vague, Assumption with a stated ceiling |
| "reliable", "robust" | Failure modes undefined | Open Question if retry/idempotency design is material; Assumption with standard retry-with-backoff otherwise |
| "maybe", "perhaps", "could" | Decision deferred | Resolve to a concrete choice — Assumption if default is safe, Open Question if architectural |
| "later", "eventually", "in v2" | Scope creep warning | Move to Non-goals explicitly |
| "like X does" (X is another system) | Unstated parity expectation | Ask which specific behaviours of X are required. Do not assume full parity |
| "the usual way" | Convention inferred | Look up the project default in CLAUDE.md and record explicitly as "project default — see <ref>" |
| "supports A, B, etc." | Enumeration incomplete | Ask for the full list; forbid "etc." in the final draft |
| "handles X" | Steps elided | Ask for the specific steps taken on X |

## Structural patterns

| Pattern | Action |
|---|---|
| Entity mentioned but no lifecycle stated | Ask the lifecycle question explicitly |
| Action mentioned but no trigger stated | Ask who or what initiates it |
| Action mentioned but no failure mode stated | Ask: what errors can this produce, and what should happen on each? |
| Cross-cutting row left empty after interview | Revisit with a direct one-liner: "Does <concern> apply? Yes, No, or project default?" |
| Performance target stated for some endpoints but not others | Ask whether the unlisted endpoints inherit the same target or have different ones |
| Multi-tenancy mentioned but tenant boundary not named | Ask: is tenancy enforced by subject, column, schema, or row-level security? |
| Migration implied but rollback not discussed | Ask: if we ship this and it goes wrong in prod, what does the rollback look like? |
| External service named but SLA / failure handling not discussed | Ask: what happens when the external service is down or slow? |

## Resolvable by project default

Consult the repo's `CLAUDE.md` and existing `docs/design/` / `docs/spec/` for prior decisions. Record as "project default — see <ref>" rather than inventing a new policy. Defaults commonly found in this repo:

- JSON logging with correlation IDs.
- Error envelope shape.
- Rate limit (20 req/s/IP default per the Echo setup).
- Auth middleware identifier (`lib.UserAuth` or `lib.NewAuth`).
- Queue group name for NATS subscribers (`"insight"`).
- Multi-tenant routing encoded in NATS subjects (`tenant.*.site.*.appliance.*...`).

If a new design contradicts a project default, escalate to Open Question, not Assumption — the divergence is architectural.

## Deciding Assumption vs Open Question

Use this test:

- If two reasonable engineers would almost certainly pick the same default → **Assumption**.
- If two reasonable engineers might pick different defaults, and the choice changes the architecture (data model, interface shape, failure handling) → **Open Question**, marked **blocking**.
- If the user specifically says "I don't know and it matters" → **Open Question**, marked blocking.
- If the user says "I don't know, but you pick something safe" → **Assumption**, clearly labelled.

Never split the difference by writing vague prose in the body. Every ambiguity goes into one of the two tables, never into narrative.
