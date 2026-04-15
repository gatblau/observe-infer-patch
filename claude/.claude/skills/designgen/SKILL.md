---
name: designgen
description: Produce a human-authored-quality design document in docs/design/ from a problem statement, shaped to exactly what specgen expects as input. Interviews the user to surface ambiguities, domain nouns, system actions, and cross-cutting concerns, then drafts a structured Markdown design doc. Trigger on phrases like "design doc for X", "start a new design", "designgen", "draft design for <feature>", or when the user asks to begin a new feature with a design document. Do NOT trigger on requests to edit or review an existing design.
---

# designgen — Problem Statement to Design Document

Produce a design document in `docs/design/` that, when fed to `/specgen`, yields high-quality specs with minimal Phase 1B open questions. The skill is an interviewer + drafter: it asks targeted questions to surface what a good design doc needs, then writes the draft.

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Arguments

- `topic` (required): short feature or problem name, used as the filename seed. E.g. `nats-key-rotation`, `appliance-telemetry`.
- `out-dir`: default `docs/design/`.
- `mode`: `interview` (default — ask questions, then draft) | `draft` (skip questions, draft directly from the prompt if the user provides enough context inline).

## Output

Single Markdown file at `docs/design/<topic>.md`. Sections are shaped to match what `specgen` extracts (Phase 1 glossary, Phase 2 components, Phase 3 actions, Phase 4 cross-cutting). See `references/design-template.md`.

If the file already exists: stop and ask whether to overwrite, extend, or pick a new topic name. Never overwrite silently.

## Active mode

**Observation → Patch (single new file).** Phases 1–3 are Observation: interview the user, surface ambiguities, do not invent. Phase 4 drafts content captured from the interview. Phase 5 is the patch-mode verification gate before writing the new file. Never overwrites — if the file exists, stop and ask.

## Process

### Phase 1 — Intake (always)
Ask the user to describe, in a few sentences each:
- **What problem is being solved.** What's broken or missing today.
- **Who or what triggers the work.** User action, event, schedule, external system.
- **What observable result is expected.** What changes after the feature exists.
- **Known constraints.** Non-negotiables — tech stack, backward compatibility, compliance, deadlines.
- **Out of scope.** What this explicitly will not address.

If any answer is missing or thin, ask one follow-up question at a time — do not dump a questionnaire. Stop when you have enough to draft, not when you have "everything".

### Phase 2 — Targeted interview (mode: interview only)
Load `references/interview-questions.md`. Ask questions in each of these five areas, skipping any already covered in Phase 1:

1. Domain model — nouns, their relationships, their lifecycle.
2. Actions — the verbs the system performs; triggers, inputs, outputs, side effects.
3. Non-functional — performance targets, scale, availability, security posture.
4. Cross-cutting — auth, logging, metrics, tracing, rate limiting, pagination, CORS, health, migrations, graceful shutdown. Which apply? Any project-specific policy the design needs to record?
5. Known unknowns — things the user is aware they don't know yet.

Keep to one question per turn when the answer needs thinking. Batch simple yes/no questions.

### Phase 3 — Ambiguity sweep
Load `references/ambiguity-patterns.md`. Re-read the gathered answers. For each ambiguity pattern detected, decide:
- Resolvable by a reasonable default → record as **Assumption** in the draft.
- Materially affects architecture → record as **Open question** in the draft.
- Still unclear → ask the user one more time.

This is the step that most raises the quality of the downstream spec. Do not skip.

### Phase 4 — Draft
Load `references/design-template.md`. Produce the full document. Every section the template lists must be present — if a section genuinely does not apply, write a one-line "N/A — <reason>" rather than omitting. British English throughout.

### Phase 5 — Self-check before writing the file
Verify:
- Every noun introduced has an entry in the Domain Model section.
- Every action described has inputs, outputs, and side effects named.
- Every cross-cutting concern has an explicit stance (applies / does not apply / project-default).
- Every assumption is labelled.
- Every open question is labelled and marked blocking / non-blocking.
- File name matches `docs/design/<topic>.md` with kebab-case.

Only then write the file. After writing, show the user the file path and a one-paragraph summary of what was drafted, plus the count of assumptions and open questions. Suggest next step: "Run `/specgen phase: 1-2` against `docs/design/` to produce initial analysis and architecture."

## References (load on demand)

- `references/design-template.md` — full section template. Load at start of Phase 4.
- `references/interview-questions.md` — topical questions organised by area. Load at start of Phase 2.
- `references/ambiguity-patterns.md` — common ambiguity flags and how to resolve them. Load at start of Phase 3.

## What designgen does not do

- Does not produce specs (that is `/specgen`).
- Does not produce code or pseudocode.
- Does not edit existing design documents (use a direct edit instead — this skill is for new designs).
- Does not decide architecture. It captures the user's thinking in a form that makes downstream spec generation crisp.
