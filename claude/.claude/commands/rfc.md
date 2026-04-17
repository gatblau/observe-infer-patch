---
description: Capture a Request for Change as a structured document (problem, current vs desired behaviour, scope, acceptance criteria) in observation-mode. Produces input suitable for `/breakdown`. Writes issue-xxx-rfc.md when the RFC is large.
argument-hint: <paste change request text, or path to a saved file, plus optional issue id>
---

# /rfc — Request for Change

You have been given the following change request as input:

```
$ARGUMENTS
```

If `$ARGUMENTS` is empty, ask the user to either paste the change request (feature idea, bug description, refactor motivation, customer ask) or provide a path to a saved file, plus optionally an issue id like `issue-123`. Stop until they reply.

If `$ARGUMENTS` looks like a file path, Read it. Otherwise treat the argument text as the change request itself.

## Purpose

An RFC is the **upstream input** to `/breakdown`. It states what should change and why, anchored in repository evidence, without prescribing an implementation. Think of it as: everything `/breakdown` needs to decompose the work into phases, minus the phase decomposition itself.

Accepted sources that commonly become an RFC:
- **Feature request or enhancement idea** — user-facing capability that does not yet exist.
- **Bug report** — observed behaviour that diverges from intended behaviour.
- **Refactor motivation** — non-functional improvement (performance, maintainability, security posture).
- **Customer-reported symptom** — a narrative that must be converted into concrete acceptance criteria.
- **Design intent delta** — a shift in requirements captured verbally or in chat, not yet written down.

Do **not** use this command to analyse a stack trace (that is `/rca`) or to decompose an already-structured diagnosis into phases (that is `/breakdown`).

## Mode

Operate strictly in **Observation mode** as defined in `CLAUDE.md` (Working modes → Observation mode). This command produces a **document**, not a plan and not a patch. Do not edit source files. Do not invent unverified identifiers, signatures, or schema objects. Separate what the user asked for from what the repository currently does from what is assumed.

If the change touches `schema/**` or `*.sql`, layer **SQL & schema safety** rules on top — never name tables/columns/functions as verified unless visible in the repo.

## Procedure

1. **Restate the request.** In one or two sentences, restate what the user is asking for in your own words. If the request is ambiguous (two plausible interpretations, missing target, unclear success condition), list the interpretations and ask the user which one to RFC against; do not silently pick one.
2. **Anchor in the repository.** For every concrete signal in the request (file path, symbol, route, table, env var, config key, feature name, error string), verify whether it currently exists. Use Grep/Glob/Read. Record exact `path:line` for anything found; record "not found" for anything not found — that is itself evidence. Batch the signal lookups in parallel rather than sequentially.
3. **Capture current behaviour.** Describe what the repository does today in the area the change would touch, citing `path:line`. Do not editorialise. If the current behaviour cannot be determined from the repo alone (e.g. depends on runtime data, external service, user configuration), say so under **Open questions**.
4. **Capture desired behaviour.** State the target state as an outcome a future reader can test against, not as an implementation. Prefer "request X returns Y under condition Z" over "add function foo in bar.go".
5. **Define scope boundaries.** Make **In scope** and **Out of scope** explicit. Call out adjacent areas that are tempting but not part of this change, with a one-line reason.
6. **Draft acceptance criteria.** Each criterion must be **objective and runnable**: an HTTP/CLI scenario, a specific test name, a schema invariant, a log/metric signal, or a user-visible behaviour. Vague criteria ("feels faster", "is cleaner") are banned — convert them into measurable equivalents or move them to **Open questions**.
7. **List constraints.** Non-functional requirements and boundaries `/breakdown` must respect: backward compatibility, performance budget, security posture, deployment ordering, deprecation windows, supported clients, SLAs.
8. **Offer a proposed direction, not a plan.** A single paragraph of high-level approach hints is fine if it helps downstream understanding. Do **not** enumerate phases, files to edit, or function names — that is `/breakdown`'s job. If unsure, leave this section empty rather than speculate.
9. **Collect open questions.** Every unresolved ambiguity that would block `/breakdown` from producing a useful plan. Flag items as **blocking** when `/breakdown` cannot safely proceed without an answer.
10. **Decide on file output.** If the RFC has more than ~8 acceptance criteria, touches more than ~3 concrete files/packages/schema objects, or runs longer than ~200 lines, write it to `issue-<id>-rfc.md` at the repo root. Otherwise return inline. The issue id comes from `$ARGUMENTS` if provided; else ask the user for one (do not invent it).

## Required output structure

If writing to a file, the file content uses this same structure. Always also reply inline with: the file path written (if any), a one-paragraph summary, and the acceptance-criteria list so the user can review without opening the file.

### Title
- One-line title of the change. Prefix with the issue id when known (e.g. `issue-123: …`).

### Restated request
- Your one-or-two-sentence restatement of what the user is asking for. Note any interpretation choices you had to make and which one you went with.

### Motivation
- Why this change matters. Anchor in the original request. Do not invent business context that was not stated.

### Current behaviour
- What the repository does today in the affected area, with `path:line` citations. Mark anything you could not verify as **unverified**.

### Desired behaviour
- The target state as an observable outcome. No implementation detail. No file names unless the user specified them.

### Scope
- **In scope:** what this RFC intends to change.
- **Out of scope:** adjacent work this RFC will not touch (one-line reason each).

### Acceptance criteria
- Numbered, objective, runnable criteria. Each item should be directly testable (named test, HTTP/CLI scenario, schema invariant, metric/log signal, user-visible behaviour).

### Constraints
- Non-functional requirements `/breakdown` must respect (compatibility, performance, security, rollout, deprecation). Omit the section if genuinely none apply — do not pad.

### Proposed direction (optional)
- One paragraph of high-level approach hints, or omit. Must not prescribe phases, files, or symbols.

### Open questions
- Numbered ambiguities that remain. Flag each as **blocking** or **non-blocking** for `/breakdown`. A blocking item means `/breakdown` should not run until the user resolves it.

### Handoff to /breakdown
- One-line suggested invocation, e.g. `/breakdown issue-123-rfc.md issue-123`. If any open question is blocking, state that the user should resolve it first.

## File-writing rules

- Filename: `issue-<id>-rfc.md` at the repo root. If the user gave no id, ask; do not fabricate one.
- Use the Write tool. Do not overwrite an existing file with the same name without confirming with the user first.
- After writing, reply with: the absolute path, the one-paragraph summary, the acceptance-criteria list, and any **blocking** open questions so the user can decide whether to resolve them before invoking `/breakdown`.