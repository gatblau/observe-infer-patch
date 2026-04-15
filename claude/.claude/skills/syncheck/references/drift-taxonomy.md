# Drift taxonomy

Every finding carries exactly one of four labels. The label dictates who resolves the drift and with which skill.

## INTENT-DRIFT

**Definition:** The design document has been edited since the spec was generated, and the spec (and therefore the code) no longer reflects current human intent.

**Signal:** A design file's `git log -1 --format=%ct` is newer than the corresponding spec file(s), and the design content contains nouns/actions/decisions not present in the spec.

**Resolution playbook:**
1. Human reviews the design delta to confirm it is intentional (not a typo, not an exploratory note).
2. Run `/specgen phase: 1-2` to refresh analysis and architecture — new assumptions or open questions may surface.
3. Human resolves any new open questions.
4. Run `/specgen phase: 3-6` scoped to affected components.
5. Run `/codegen step: "<N>: <Component>"` for each component with a changed spec.
6. Re-run `/syncheck` to confirm zero remaining drift.

## CONTRACT-DRIFT

**Definition:** A spec has been edited and the code no longer matches the contract. The design is still the source of truth; the spec still reflects it; only the code is behind.

**Signal:** A Phase 3 or Phase 4 spec's `git log -1 --format=%ct` is newer than the source files it owns, and a contract aspect (signature, JSON tag, route, error code, env var, DDL, metric name) differs between spec and code.

**Resolution playbook:**
1. Confirm no REALITY-DRIFT findings exist on the same file (if both, treat as REALITY-DRIFT — the code change may be intentional).
2. Run `/codegen step: "<N>: <Component>"`.
3. Re-run `/syncheck` to confirm the finding is gone and no regressions appeared in adjacent components.

## REALITY-DRIFT

**Definition:** The code has been edited (hotfix, emergency patch, refactor) and the spec no longer describes what the code does. The code, not the spec, is the current truth.

**Signal:** A source file's `git log -1 --format=%ct` is newer than the spec file that owns it, and a contract aspect differs.

**Resolution playbook (human decision required):**
Present **both** alternatives, never pick one:

- **Option A — Revert code to spec.** Produce a diff that reverts the drifted aspects. Use when the code change was accidental, an unreviewed experiment, or violated the contract without authorisation.
- **Option B — Update spec to match code.** Produce a diff against the Phase 3 or Phase 4 spec file. Use when the code change was a legitimate decision that simply never made it back into the spec. Must also:
  - Update any Phase 1 assumption affected.
  - Update Phase 2 Shared Types or Configuration if the change touches them.
  - Re-run `/specgen` Phase 6 self-audit against the edited specs.

Never auto-regenerate code on a REALITY-DRIFT finding — doing so destroys the hotfix.

## SCOPE-GAP

**Definition:** An artefact exists in one layer with no counterpart in the layers that should contain it. Four concrete sub-cases:

| Sub-case | Example |
|---|---|
| Design orphan | Design mentions a concept with no Phase 1 glossary entry, no Phase 3 component, and no code |
| Spec orphan | Phase 3 component spec exists but no source file at its declared `**File:**` path |
| Code orphan | Source file exports symbols that no Phase 3 or Phase 4 spec owns |
| Migration orphan | `migrations/*.sql` file with no matching Phase 3 Data Model section, or Data Model with no migration |

**Resolution playbook (human triage):**
For each orphan, present three options and let the human pick:

1. **Fill the gap** — add the missing counterpart (new spec section, new code, new migration).
2. **Delete the orphan** — if it is dead or experimental.
3. **Mark as intentional** — record as a Phase 1 Assumption (e.g. "Component X is externally owned; no spec is produced here"). Future syncheck runs will ignore it once the assumption is present.

## Likely-source heuristic

When classifying, compare modification times via `git log -1 --format=%ct <file>`:

- Spec newer than code → lean CONTRACT-DRIFT.
- Code newer than spec → lean REALITY-DRIFT.
- Design newer than spec → INTENT-DRIFT.
- Uncommitted changes on either side → treat the uncommitted side as "newer".

This is a hint, not a verdict. Always phrase the report as "likely source: <layer>", never "source: <layer>". The human is the final arbiter.
