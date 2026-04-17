---
description: Apply a single phase of a plan (or an in-context plan/RCA) using patch-mode. Stops at that phase's exit criteria.
argument-hint: [plan path or issue id] <phase number> (e.g. issue-123-plan.md 2, or just "2" to use an in-context plan/RCA)
---

# /apply — Execute One Plan Phase

You have been given the following input:

```
$ARGUMENTS
```

Parse `$ARGUMENTS` to extract:
- **Plan source:** a path to a plan file (e.g. `issue-123-plan.md`), or an issue id (e.g. `issue-123` / `123`) which resolves to `issue-<id>-plan.md` at the repo root. **Optional** — if omitted, fall back to the most recent plan or RCA produced in the current conversation context (see "In-context fallback" below).
- **Phase number:** the phase to implement (integer). Required.

### In-context fallback

If no plan path or issue id is supplied:

1. Scan the current conversation for the most recent artefact that looks like either:
   - a **plan** (phased structure with explicit phases, exit criteria, and open assumptions — e.g. produced by `/plan` or similar), or
   - an **RCA** (root-cause analysis with ranked hypotheses — e.g. produced by `/rca`).
2. If exactly one such artefact exists, use it as the plan source. State clearly in the output which artefact you selected and that it came from conversation context (not a file).
3. If the selected artefact is an **RCA** rather than a phased plan, it has no numbered phases. In that case:
   - Treat the top-ranked hypothesis's **What to verify next** / remediation as a single implicit "Phase 1".
   - Only proceed if the user supplied phase number is `1`. Otherwise stop and ask the user to either supply a real plan file or confirm they want Phase 1 of the RCA-derived fix.
   - Require that the top hypothesis is labelled **high confidence** in the RCA. If it is medium/low, stop and tell the user to verify the hypothesis first (per Inference mode rules) before patching.
4. If zero or multiple candidate artefacts are present in context, stop and ask the user to disambiguate — do not guess.

If the phase number is missing or ambiguous, ask the user once and stop. Do not guess the phase. Do not pick the "next" phase on your own.

## Mode

Operate strictly in **Patch mode** as defined in `CLAUDE.md` (Working modes → Patch mode). If the phase touches `schema/**` or `*.sql`, layer **SQL & schema safety** on top.

This command applies **exactly one phase**. Do not start the next phase, even if it looks trivial. The user will re-invoke `/apply` for subsequent phases.

## Procedure

1. **Load the plan.** If a plan path or issue id was supplied, read the plan file; if it does not exist, stop and tell the user. If no plan source was supplied, use the in-context fallback above and treat the selected conversation artefact as the plan.
2. **Locate the phase.** Find the section for the requested phase number. If it does not exist, list the available phases and stop. For an RCA-derived fallback, the only valid phase is `1` (the remediation for the top hypothesis).
3. **Re-verify preconditions.** For every file, symbol, signature, route, config key, env var, schema object, or external interface the phase says it will change, confirm it currently exists in the repo with Read/Grep. Note any drift from what the plan claimed. If a precondition drift resembles a CONTRACT-DRIFT finding (spec says X, code says Y, and the plan aligns with the spec), proceed with the patch as planned. If it resembles REALITY-DRIFT (code was recently changed and the plan would revert that change), stop and surface it — the plan may be stale.
4. **Check open assumptions.** Re-read the plan's **Open assumptions** section. If any assumption marked **blocking** (or any assumption this phase depends on) is still unverified, stop and surface it — do not patch.
5. **Respect prior phases.** If earlier phases were prerequisites and their changes are not present in the repo, stop and tell the user which earlier phase appears unimplemented.
6. **Apply the smallest viable patch.** Make only the edits this phase requires. Match existing project conventions. No opportunistic refactors, renames, or "while I'm here" cleanups. No new helpers/abstractions unless the plan calls for them and they are verified.
7. **Run the exit criteria.** Execute the exact commands listed under the phase's **Exit criteria** (e.g. `go build ./...`, the specified `go test -run ... ./<pkg>` invocations, `make test`/`make lint`/`make integration-test` if listed, schema scripts if the phase touches SQL). Capture results.
8. **If a check fails:** diagnose the root cause, fix it within the same phase if it is in-scope, and re-run. Do not weaken or skip the criteria. Do not `--no-verify` past hooks. If the failure is out of scope for this phase, stop and report.
9. **Do not commit** unless the user explicitly asked. Patch mode produces diffs and runs checks; commit/PR is a separate step.

## Required output structure

### Phase being implemented
- Phase number, title, and the plan source (file path, or "in-context plan" / "in-context RCA (top hypothesis)" if using the fallback).

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
- One of: **Phase {n} complete — exit criteria met** / **Phase {n} blocked — see Assumptions/Exit criteria results**. If complete, point the user at the next phase number to invoke (`/apply <plan> <n+1>`) without starting it.