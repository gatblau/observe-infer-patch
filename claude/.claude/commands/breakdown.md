---
description: Break a structured diagnosis (RCA, syncheck report, specgen Playbook excerpt, design delta, issue description) into an incremental implementation plan, in inference-mode. Writes issue-xxx-plan.md when the plan is large.
argument-hint: <paste diagnosis text, or path to a saved diagnosis file, plus optional issue id>
---

# /breakdown — Decompose a Diagnosis into a Phased Plan

You have been given the following diagnosis as input:

```
$ARGUMENTS
```

If `$ARGUMENTS` is empty, ask the user to either paste a diagnosis or provide a path to a saved file (and optionally an issue id like `issue-123`). Stop until they reply.

If `$ARGUMENTS` looks like a file path, Read it. Otherwise treat the argument text as the diagnosis itself.

## Accepted input types

The breakdown procedure is input-agnostic. Typical inputs:

- **`/rca` output** — ranked root-cause hypotheses with evidence.
- **`/syncheck` report** — CONTRACT-DRIFT, INTENT-DRIFT, REALITY-DRIFT, or SCOPE-GAP findings to close.
- **`/specgen` Phase 5 Playbook excerpt** — a subset of components to implement.
- **Design-doc delta or git diff** — describing intent change that needs a code response.
- **Plain-text issue description** — bug report, feature request, customer-reported symptom.

Anchor on the named evidence in the input, verify it against the repo, then decompose into phases. **Verification rigour scales with input quality:** RCA and syncheck outputs carry concrete signals (file paths, symbol names, drift labels); plain-text issues carry few, so more preconditions will end up under **Open assumptions** and the plan will be gated on verifying them before Phase 1.

## Mode

Operate in **Inference mode** as defined in `CLAUDE.md` (Working modes → Inference mode). This command produces a **plan**, not a patch. Do not edit source files. Do not invent unverified identifiers, signatures, or schema objects. If the input's top claim (RCA hypothesis, syncheck finding, etc.) is not yet verified against the repo, say so and gate the plan on verification.

If any planned step touches `schema/**` or `*.sql`, layer **SQL & schema safety** rules on top.

## Procedure

1. **Anchor on the diagnosis.** Identify the specific claim to plan against: top-ranked RCA hypothesis, set of syncheck findings to close, Playbook steps to implement, intent change to respond to, or requirement from the issue text. If multiple candidates are tied or ambiguous, ask the user which to plan against, or plan against the first and flag the assumption.
2. **Verify before planning.** For every file, symbol, route, table, function, or env var the plan will touch, confirm it exists by reading the repo. List anything you could not verify under **Open assumptions** — the plan must not silently depend on them.
3. **Decompose into incremental phases.** Each phase must be independently shippable (compiles + relevant tests pass) and ordered so that earlier phases reduce risk for later ones. Prefer small phases over a single big bang.
4. **Define exit criteria per phase.** Exit criteria are objective and runnable: which `go build` succeeds, which test commands pass (`go test -run ... ./<pkg>`), which `make` targets are green (`make test`, `make lint`, `make integration-test` when relevant), and any schema/migration verification.
5. **Decide on file output.** If the plan has more than ~3 phases, more than ~30 actionable steps, or spans multiple packages/schema files, write it to `issue-<id>-plan.md` at the repo root. Otherwise return inline. The issue id comes from `$ARGUMENTS` if provided; else ask the user for one (do not invent it).

## Required output structure

If writing to a file, the file content uses this same structure. Always also reply inline with: the file path written, a one-paragraph summary, and the phase list with exit criteria.

### Anchor (from input)
- One-line restatement of the specific claim being planned against (root cause, drift finding, component to build, intent change, or requirement) and a pointer to the supporting evidence in the input.

### Verified context
- Files, symbols, routes, schema objects you confirmed exist, each with `path:line`.

### Open assumptions
- Anything the plan depends on that is not yet verified. If any item here would block safe implementation, flag it as **blocking** and recommend the user verify before starting Phase 1.

### Scope
- **In scope:** what this plan will change.
- **Out of scope:** what it explicitly will not touch (and why, if non-obvious).

### Phases
For each phase (numbered, in order):

#### Phase {n} — {short title}
- **Goal:** one sentence.
- **Changes:** bullet list of intended edits, each naming the exact file (and symbol where known). Mark unverified targets clearly.
- **Rationale:** why this phase exists and why it comes here in the order.
- **Exit criteria** (all must hold to move on):
  - Code compiles: `go build ./...` (or the narrower package build if that's all that changed).
  - Tests pass: list the exact `go test` invocations (e.g. `go test -run TestX ./route`) and any `make` targets (`make test`, `make lint`, `make integration-test` when applicable).
  - Schema/migration checks (only if the phase touches `schema/**` or `*.sql`): which scripts run cleanly, rollback considerations, what was verified visually in the SQL file.
  - Behavioural check: a one-line manual or scripted verification of the fix's effect for this phase.
- **Risks / rollback:** what could regress and how to undo this phase in isolation.

### Cross-cutting validation
- Final end-to-end check after all phases (e.g. `make check`, `make integration-test`, specific HTTP/NATS scenarios).

### Follow-ups (not in this plan)
- Things noticed during planning that should become separate work.

## File-writing rules

- Filename: `issue-<id>-plan.md` at the repo root. If the user gave no id, ask; do not fabricate one.
- Use the Write tool. Do not overwrite an existing file with the same name without confirming with the user first.
- After writing, reply with: the absolute path, the one-paragraph summary, and the phase titles + exit-criteria one-liners so the user can review without opening the file.