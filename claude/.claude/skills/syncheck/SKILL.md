---
name: syncheck
description: Detect and classify drift between design documents (docs/design/), specifications (docs/spec/), and source code in any language. Read-only report by default; proposes reconciliation patches only in fix mode with explicit user approval. Invoke explicitly — trigger only on imperative phrases like "run syncheck", "check drift", "verify specs match code", "reconcile design spec code", or /syncheck. Do NOT trigger on passive questions like "is the spec up to date".
---

# syncheck — Design ↔ Spec ↔ Code drift detection

Detect where the three artefact layers disagree, classify each finding so the right resolution path is obvious, and propose (never auto-apply) the minimum patch that closes the gap. Code inspection only — never executes the code under test.

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Active mode

**Observation.** Read-only across design, spec, and code. Tier-tagged extraction (T1–T4) is the observation-mode "verified vs inferred" axis: low-tier findings are flagged "regex-confidence — verify manually". Even `mode: fix` proposes patches and stops for explicit user approval — never writes autonomously.

## Layers

| Layer | Location | Source of |
|---|---|---|
| Design | `docs/design/*.md` | Human intent |
| Spec | `docs/spec/phase-*.md` (+ optional `docs/spec/components/*.md`) | Machine-consumable contract |
| Code | Repository source tree + `migrations/*.sql` | Implementation |

Design ↔ Code is never compared directly. Specs are the only reconciliation layer — if they are missing, the skill stops and says so.

## Arguments

- `scope`: `all` (default) | `design-spec` | `spec-code` | `<component-name>`
- `mode`: `report` (default, read-only) | `fix` (proposes patches; still writes nothing until approved)
- `since`: optional git ref. When set, only files changed since that ref are inspected. Use for fast incremental checks on large repos.

## Drift taxonomy (always use these four labels)

Load `references/drift-taxonomy.md` at the start of every run for full definitions and resolution playbooks. Headline:

| Label | Meaning | Owner |
|---|---|---|
| INTENT-DRIFT | Design changed; spec/code stale | Human (may create open questions) |
| CONTRACT-DRIFT | Spec changed; code stale | `/codegen` re-run for affected component |
| REALITY-DRIFT | Code changed; spec no longer describes truth | Human decides: revert code **or** update spec |
| SCOPE-GAP | Artefact exists in one layer with no counterpart elsewhere | Human triage |

REALITY-DRIFT must never be auto-fixed by regenerating code — regeneration would destroy the hotfix. The skill surfaces two alternative diffs (code revert vs spec edit) and stops.

## Process

### Step 1 — Toolchain probe
Load `references/extraction-tiers.md`. Run `which tree-sitter`, `which go`, `which tsc`, `which python3`, `which cargo` (fail-soft). Record which tiers are available. Report header must state this. Every finding is tagged with the tier that produced it.

### Step 2 — Inventory
- Design files: `Glob docs/design/**/*.md`.
- Spec components: read `docs/spec/phase-2-architecture.md` → Component Inventory table. For each component, read its Phase 3 spec (declared `**File:**` path used in Step 3). Also read Phase 4 cross-cutting specs.
- Source files: `Glob` with the default allowlist (see `references/extraction-tiers.md` for full list). Excludes: `vendor/`, `node_modules/`, `target/`, `dist/`, `build/`, `.git/`, `docs/`, generated directories. Overridable via `.syncheck.yml` or the repo's `CLAUDE.md`.

If `docs/spec/` is missing or empty: stop. Tell the user to run `/specgen` first.

### Step 3 — Extract contracts
Load `references/design-spec-checks.md` and `references/spec-code-checks.md`.

For each spec component, apply the highest-tier extractor available against the file path declared in the spec's `**File:**` field. If the declared file does not exist, emit a `SCOPE-GAP` finding and continue with name-based fallback matching.

Extraction is confidence-tagged (T1 filesystem / T2 regex / T3 tree-sitter / T4 native AST). Low-tier findings are marked "regex-confidence — verify manually" in the report.

### Step 4 — Diff & classify
For each (spec aspect, code aspect) pair:
- If equal, no finding.
- If different, classify via the taxonomy. Use `git log -1 --format=%ct` on each artefact as a tiebreaker for CONTRACT vs REALITY: most-recently-modified layer is the **likely** drift source. This is a hint, not a verdict — the report states "likely source: <layer>" and lets the human adjudicate.

### Step 5 — Report (always produced)
Load `references/report-template.md`. Output: header (toolchain probe results, scope, timestamp, git HEAD), summary counts per label, then a findings table grouped by component. One row per finding with: component · aspect · expected · actual · label · tier · likely source · suggested action.

Write the report to stdout in the response. Do not write any file unless the user explicitly asks for a saved report.

### Step 6 — Propose patches (`mode: fix` only)
For each finding:
- **CONTRACT-DRIFT** → propose `/codegen step: "<N>: <ComponentName>"` with a one-line justification of what will change.
- **INTENT-DRIFT** → propose `/specgen phase: 3-6` scoped to affected components, then `/codegen` for each.
- **REALITY-DRIFT** → present **two** alternative diffs, clearly labelled: (a) revert code to match spec, (b) edit spec to match code. Do not default to one. Do not write either until the user picks one.
- **SCOPE-GAP** → no automated patch. Print the orphan and the three possible resolutions: add the missing counterpart in the other two layers, delete the orphan, or mark it as intentional in Phase 1 Assumptions.

Wait for explicit user approval before any write.

## What this skill does not do

- Execute the code (no `go run`, no `npm start`, no container boot, no test runs).
- Infer semantic equivalence (`VAL_ERR` ≠ `ValidationError` — both are flagged; the human decides).
- Self-schedule (no cron, no hook, no watcher — every run is user-initiated).
- Regenerate artefacts directly. Only points at the existing `/specgen` or `/codegen` skill invocation that would close the gap.
- Compare design directly to code. Design ↔ Spec and Spec ↔ Code are the only axes.

## References (load on demand)

- `references/drift-taxonomy.md` — four labels, definitions, resolution playbooks. Load at start of run.
- `references/extraction-tiers.md` — T1–T4 probing, file-extension allowlist, exclude rules, per-language tree-sitter grammar names. Load at start of Step 1.
- `references/design-spec-checks.md` — design↔spec aspect checks (glossary, assumptions, noun/action/cross-cutting coverage). Load at start of Step 3 if scope includes `design-spec`.
- `references/spec-code-checks.md` — spec↔code aspect checks (routes, signatures, JSON tags, env vars, error codes, metric names, DDL, middleware references). Load at start of Step 3 if scope includes `spec-code`.
- `references/report-template.md` — findings table format, header fields, summary counts. Load at start of Step 5.
