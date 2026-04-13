---
description: Implement a single phase of an issue-xxx-plan.md using patch-mode. Stops at that phase's exit criteria.
argument-hint: <plan path or issue id> <phase number> (e.g. issue-123-plan.md 2)
---

# /patch — Implement a Plan Phase

You have been given the following input:

```
$ARGUMENTS
```

Parse `$ARGUMENTS` to extract:
- **Plan source:** a path to a plan file (e.g. `issue-123-plan.md`), or an issue id (e.g. `issue-123` / `123`) which resolves to `issue-<id>-plan.md` at the repo root.
- **Phase number:** the phase to implement (integer).

If either is missing or ambiguous, ask the user once and stop. Do not guess the phase. Do not pick the "next" phase on your own.

## Mode

Operate strictly in **Patch mode** as defined in `CLAUDE.md` (Working modes → Patch mode). If the phase touches `schema/**` or `*.sql`, layer **SQL & schema safety** on top.

This command implements **exactly one phase**. Do not start the next phase, even if it looks trivial. The user will re-invoke `/patch` for subsequent phases.

## Procedure

1. **Load the plan.** Read the plan file. If it does not exist, stop and tell the user.
2. **Locate the phase.** Find the section for the requested phase number. If it does not exist, list the available phases and stop.
3. **Re-verify preconditions.** For every file, symbol, signature, route, config key, env var, schema object, or external interface the phase says it will change, confirm it currently exists in the repo with Read/Grep. Note any drift from what the plan claimed.
4. **Check open assumptions.** Re-read the plan's **Open assumptions** section. If any assumption marked **blocking** (or any assumption this phase depends on) is still unverified, stop and surface it — do not patch.
5. **Respect prior phases.** If earlier phases were prerequisites and their changes are not present in the repo, stop and tell the user which earlier phase appears unimplemented.
6. **Apply the smallest viable patch.** Make only the edits this phase requires. Match existing project conventions. No opportunistic refactors, renames, or "while I'm here" cleanups. No new helpers/abstractions unless the plan calls for them and they are verified.
7. **Run the exit criteria.** Execute the exact commands listed under the phase's **Exit criteria** (e.g. `go build ./...`, the specified `go test -run ... ./<pkg>` invocations, `make test`/`make lint`/`make integration-test` if listed, schema scripts if the phase touches SQL). Capture results.
8. **If a check fails:** diagnose the root cause, fix it within the same phase if it is in-scope, and re-run. Do not weaken or skip the criteria. Do not `--no-verify` past hooks. If the failure is out of scope for this phase, stop and report.
9. **Do not commit** unless the user explicitly asked. Patch mode produces diffs and runs checks; commit/PR is a separate step.

## Required output structure

### Phase being implemented
- Phase number, title, and the plan file path.

### Verified preconditions
- Each precondition the phase depends on, with `path:line`. Note any drift from the plan and how you handled it.

### Assumptions
- Anything still unverified. If blocking, **stop here** and explain. Otherwise list and proceed.

### Changes applied
- For each edited/created/deleted file: path and one-line description of the edit and why. Reference the plan's bullet that motivated it.

### Exit criteria results
- Each command from the phase's exit criteria, with pass/fail and a short excerpt of failure output if any. Include the behavioural check result.

### Risks and follow-ups
- Side effects, compatibility concerns, or items the plan deferred to later phases that this implementation surfaced. Do not act on them.

### Status
- One of: **Phase {n} complete — exit criteria met** / **Phase {n} blocked — see Assumptions/Exit criteria results**. If complete, point the user at the next phase number to invoke (`/patch <plan> <n+1>`) without starting it.