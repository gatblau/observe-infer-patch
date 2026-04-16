# Using `.claude/` — a layman's guide

This folder configures Claude Code for this repository. You don't need to read any of the rule or skill files yourself — Claude reads them. What you need to know is **which words to type in your prompt** to get the right behaviour for the activity you're doing.

The guide below is organised by software development lifecycle (SDLC) activity. Find your activity, copy the example prompt, adapt it.

---

## How the configuration is organised

Three kinds of asset live under `.claude/`:

| Kind | Where | What it does | When it activates |
|---|---|---|---|
| **Rules** | `.claude/rules/` | Always-on defaults that condition every Claude reply | Automatically, every message |
| **Commands** | `.claude/commands/` | Short procedures invoked by typing `/<name>` | When you type the slash command |
| **Skills** | `.claude/skills/` | Multi-phase workflows triggered by natural language or `/<name>` | When your prompt matches the trigger phrases |

You never have to manage rules. Commands and skills are the things you actively use.

### What rules give you for free

Whenever you type **anything**, Claude is already operating under three "modes" and one safety "layer":

- **Observation mode** (default) — gathers evidence first, separates verified facts from inferences, doesn't invent identifiers.
- **Inference mode** — switches on when you ask "why" / "what could cause" / "how do you think".
- **Patch mode** — switches on when you explicitly ask for code changes; only edits after preconditions are verified.
- **sql-safety layer** — composes on top of any mode whenever SQL or schema work is happening; never claims a table/column exists unless it can see it.

You don't invoke these. They're the baseline.

---

## SDLC activity map

```
Idea ─► Design ─► Spec ─► Build ─► Verify ─► Release ─► Run ─► Maintain
        │         │       │        │         │           │      │
        │         │       │        │         │           │      └─ /rca, /breakdown, /apply
        │         │       │        │         │           └─ releasegen
        │         │       │        │         └─ apidiff, secscan, dbaudit
        │         │       │        └─ testgen, syncheck
        │         │       └─ codegen, schemagen
        │         └─ specgen
        └─ designgen
```

Each station below shows: **what to type**, **what happens**, **what comes next**.

---

## 1. Design — "I want to start a new feature"

**Use:** `designgen`

**Type something like:**
> Start a new design for a feature that lets operators rotate NATS keys without restarting appliances.

**What happens:** Claude interviews you in short rounds (problem, triggers, observable result, constraints, out-of-scope), runs an ambiguity sweep, then drafts `docs/design/<topic>.md` in a structure that the next stage (specgen) can consume cleanly. It will not overwrite an existing file.

**Next:** run specgen against the new design.

---

## 2. Spec — "I have a design, turn it into a buildable contract"

**Use:** `/specgen` (or natural language: "generate specs from design")

**Type:**
> /specgen phase: 1-2

…then, after you've reviewed and answered any open questions in `phase-1-analysis.md`:
> /specgen phase: 3-6

**What happens:** Claude reads everything in `docs/design/` and produces six spec documents in `docs/spec/`: assumptions, architecture, per-component contracts, cross-cutting policies, build playbook, and a self-audit. The two-round split (`1-2` then `3-6`) lets you settle ambiguities before the heavy spec generation.

**Why two rounds:** specgen is allowed to fill small gaps with reasonable defaults (recorded as assumptions) but blocking choices come back to you as questions in phase 1B.

**Next:** start building components from the playbook.

---

## 3. Build — "Implement a component from the spec"

### 3a. Application code

**Use:** `/codegen`

**Type:**
> /codegen step: "3: ProjectRepository"

The step name comes from `docs/spec/phase-5-playbook.md`. One invocation = one component (implementation + tests + wiring + migrations).

**What happens:** Claude extracts the component's contract, checks dependencies and gaps (and stops if any are blocking), generates code, runs tests with coverage, lints, then self-audits against the spec.

**If it stops with a "spec gap":** the spec is missing something needed to implement. Either edit the spec yourself or run `/specgen phase: 3-6` again.

### 3b. Database schema change

**Use:** `schemagen`

**Type:**
> schemagen — add column `email` to the `users` table, NOT NULL with backfill.

**What happens:** Claude classifies the change (additive / destructive / rename / etc.), proposes an up/down migration pair, validates it round-trip on a throwaway Postgres container, and produces a "ripple table" listing every spec, code file, and test fixture that needs to follow. Destructive changes carry an explicit rollback note.

**Next:** the ripple table will tell you exactly which `/codegen` step(s) to re-run.

---

## 4. Verify — "Make sure what we built is still consistent"

### 4a. Drift between design, spec, and code

**Use:** `/syncheck`

**Type:**
> /syncheck

Or scope it:
> /syncheck scope: spec-code

**What happens:** Claude compares the three layers and labels every disagreement as one of:
- **INTENT-DRIFT** — design moved, spec/code stale.
- **CONTRACT-DRIFT** — spec moved, code stale → re-run `/codegen`.
- **REALITY-DRIFT** — code moved (often a hotfix), spec stale → human decides whether to keep the code or revert.
- **SCOPE-GAP** — something exists in only one layer.

It never auto-fixes — even in `mode: fix` it proposes patches and waits for you to approve.

### 4b. Add tests to existing code that lacks them

**Use:** `testgen`

**Type:**
> Generate tests for `lib/nats_jwt.go` to reach 70% coverage.

**What happens:** Claude reverse-engineers the function behaviour, identifies uncovered branches and error paths, generates table-driven unit tests (and integration tests where DB/NATS are involved), runs them, reports coverage before/after.

**Important caveat:** testgen encodes **current behaviour**. If the code has a bug, the test encodes the bug. Bugs surface as `t.Skip("suspected bug…")` — fix the code via `/rca` → `/breakdown` → `/apply`, not by editing the test.

### 4c. Security posture review

**Use:** `secscan`

**Type:**
> Run a security review of `route/`.

Or scope to a single concern:
> secscan scope: secrets

**What happens:** Claude scans for hardcoded secrets, SQL injection, command injection, weak crypto, auth gaps, insecure CORS/transport, vulnerable dependencies, and sensitive logging. Findings are ranked critical/high/medium/low/info with a stated threat model. **It does not fix anything** — it produces a report you triage.

### 4d. SQL codebase critique

**Use:** `dbaudit`

**Type:**
> dbaudit

Or scope it:
> dbaudit scope: schema/v1/command/tables mode: recommendations

**What happens:** Claude statically inspects all SQL under `schema/**` (DDL, views, PL/pgSQL functions, constraints, indexes) and produces a ranked findings report classified by label (MODELLING, INTEGRITY, TYPES, PERFORMANCE, CORRECTNESS, SECURITY, COHERENCE) and severity (critical → info). Every finding cites `file:line`, states the worst-case outcome and preconditions, and carries a confidence tag. No SQL is executed; no files are modified.

**Three modes:** `report` (findings only), `recommendations` (findings + prose remediation per finding), `breakdown-ready` (findings pre-formatted for `/breakdown` → `schemagen` handoff).

**What it is not:** it does not write migrations (that's `schemagen`), detect design/spec/code drift (that's `syncheck`), audit Go call-site code, or perform a broad security scan (that's `secscan`).

### 4e. API change impact

**Use:** `apidiff`

**Type:**
> apidiff base: v1.4.2 head: HEAD

**What happens:** Claude compares the two versions across every API surface present (OpenAPI, gRPC, NATS subjects, public Go packages, public SQL views/functions). Every change is classified additive / deprecation / breaking, with a semver bump recommendation and consumer migration notes for breaking changes.

**Use this before tagging a release.**

---

## 5. Release — "Cut a version"

**Use:** `releasegen`

**Type:**
> releasegen from: v1.4.2

**What happens:** Claude reads the commit range, classifies each commit, recommends a semver bump (major/minor/patch with justification), drafts a Keep-a-Changelog fragment grouped Added / Changed / Deprecated / Removed / Fixed / Security, writes narrative release notes, and emits the suggested `git tag` / `gh release` commands as a fenced block.

**It will not tag, push, or publish.** You run the commands yourself when you're satisfied.

**Pair with apidiff** for high-confidence releases.

---

## 6. Maintain — "Something is wrong / something needs to change"

The maintenance loop is three commands working in sequence:

### 6a. Diagnose — `/rca`

**Type:**
> /rca our login endpoint started returning 500 on Monday but the error log just says "context deadline exceeded".

**What happens:** Claude switches into Inference mode, separates verified observations from hypotheses, lists ranked likely causes, and ends with **what to verify next** (specific files, log greps, queries to run). It does not patch anything.

If a `/syncheck` finding triggered the work, mention it — `/rca` knows to factor drift in.

### 6b. Plan — `/breakdown`

**Type:**
> /breakdown <paste the RCA conclusion or describe the change you want>

**What happens:** Claude turns the input into a phased implementation plan with explicit verification steps. The input doesn't have to come from `/rca` — it accepts a syncheck report, an apidiff finding, a Playbook step, a design delta, or just an issue description.

### 6c. Implement — `/apply`

**Type:**
> /apply <plan-file-or-summary> <phase-number>

**What happens:** Claude executes one phase of the plan in Patch mode — minimal local edits, verification before writing, surfaces risks. One phase per invocation so you can review between steps.

---

## Quick reference: which command/skill for which sentence

| You want to… | Use |
|---|---|
| Start a new feature with a fresh design doc | `designgen` |
| Turn a design doc into specs | `/specgen` |
| Generate code for one component from the spec | `/codegen` |
| Add or change a database table or column | `schemagen` |
| Check that design, spec, and code agree | `/syncheck` |
| Add tests to legacy code | `testgen` |
| Audit for secrets, injection, weak crypto, auth gaps | `secscan` |
| Critique SQL schema for modelling, integrity, performance, correctness | `dbaudit` |
| Compare API contracts between two versions | `apidiff` |
| Draft release notes and a version bump | `releasegen` |
| Investigate a bug or unexpected behaviour | `/rca` |
| Convert a finding/issue into a plan | `/breakdown` |
| Apply one phase of a plan as code edits | `/apply` |

---

## Three habits that make this configuration sing

1. **Stay in the chain.** Design → spec → code → drift-check is not a ceremony — each stage feeds the next. Skipping a stage means the next one has less to anchor on, and Claude has to ask you more questions or fall back on guesses.

2. **Trust the read-only skills.** `syncheck`, `secscan`, `apidiff`, `releasegen` never change files. Run them often, even speculatively — there is no destructive action to fear.

3. **Let the verification gates do their job.** When `/codegen` stops on a spec gap or `schemagen` flags a destructive change, that's the configuration working as intended. Don't try to argue past the gate — fix the upstream artefact (spec, design, scratch DB) and continue.

---

## What's NOT in this configuration

- No automated commits, pushes, tags, or PR creation. Every git-mutating action is yours to run.
- No background watchers, no cron, no hooks that fire on file save. Every workflow is explicitly invoked.
- No external API calls beyond what tools you already have (`gh`, `docker`, `psql`, etc.) make on your behalf.
- No telemetry.

If you're ever unsure what a command or skill will do, open the corresponding `SKILL.md` or `commands/*.md` file — they are short and explicit.
