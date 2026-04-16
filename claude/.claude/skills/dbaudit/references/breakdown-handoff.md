# /breakdown handoff — format (SQL audit)

`/breakdown` (see `.claude/commands/breakdown.md`) accepts a diagnosis and produces an incremental, verified implementation plan. dbaudit's `breakdown-ready` mode produces a diagnosis shaped for that consumer.

**Routing for DDL changes.** dbaudit findings that require a schema change (add constraint, change type, add index, add column, rewrite a function) must reach the repo as migrations authored by `schemagen` — not as ad-hoc edits to existing files under `schema/v1/**`. The handoff makes this explicit so each phase of the plan invokes `schemagen` as its author step, and `/apply` only as the executor.

## Why a specific shape matters

`/breakdown` anchors on "the specific claim to plan against", verifies every referenced file/symbol/route/table/function against the repo, and requires that unverified items appear under **Open assumptions**. The handoff block therefore:

- Names a single anchor claim (or a small, cohesive set of findings on the same label/component).
- Lists SQL file paths with line numbers so `/breakdown`'s verification step is cheap.
- Separates **verified** from **unverified** (anything tagged `regex-confidence` or `inferred` in the dbaudit report).
- Contains no DDL — `/breakdown` is in Inference mode and will not accept patches; DDL is `schemagen`'s output, not `/breakdown`'s input.
- States the author/executor chain: `/breakdown` (plan) → `schemagen` (migration author) → `/apply` (execute phase).

## The block

Append this at the end of the report, inside `breakdown-ready` mode only:

```markdown
---

# Handoff to /breakdown

## Anchor
{{One paragraph naming the specific claim(s) to plan against. Reference the finding ids from this report.
Example: "Fix the missing FK index on `site.tenant_id` (F-003, high, PERFORMANCE) and the
inconsistent cascade policy across children of `tenant` (F-007, high, INTEGRITY). Both findings
concentrate on the `tenant → site` boundary and should be migrated together so the cascade
change does not regress delete performance."}}

## Verified context
- `schema/v1/command/tables/tenants/tenant.sql:LLL` — tenant PK and declared children.
- `schema/v1/command/tables/locations/site.sql:LLL` — `site.tenant_id` FK, missing index.
- `schema/v1/command/tables/locations/site_status.sql:LLL` — sibling child of tenant.
- `schema/v1/manifest.json` — deploy order: `tenant` before `site` and `site_status` (verified).

## Open assumptions
- {{Item dbaudit could not fully verify. Example: "No production index stats available — the
  recommendation to add `idx_site_tenant_id` assumes `tenant` is actively updated. Confirm via
  pg_stat_user_tables on a representative environment before migration sizing."}}
- {{Flag **blocking** if the plan cannot safely start without resolving the item.}}

## Author/executor chain
- `/breakdown` — produces `issue-<id>-plan.md` with phases + exit criteria (Inference mode; no DDL).
- `schemagen` — per phase, authors the migration up/down pair. Validates against a scratch DB.
- `/apply <plan> <phase>` — executes one phase, stops at that phase's exit criteria.
- Do **not** hand-edit files under `schema/v1/command/**` — `schemagen` is the author. Hand edits
  desync the migration history and the deploy manifest.

## Invocation
Paste the following into a new message to trigger `/breakdown`:

/breakdown Findings F-003 and F-007 from dbaudit report (git HEAD <short sha>).
Diagnosis body follows.

### Diagnosis body (for /breakdown to consume)

Copy from here down into the /breakdown input:

> **Claim:** {{one-line restatement of the anchor}}
>
> **Evidence:**
> - `schema/v1/command/tables/locations/site.sql:LLL` — {{short observation}}
> - `schema/v1/command/tables/tenants/tenant.sql:LLL` — {{short observation}}
>
> **Why it matters:** {{worst-case outcome from the finding}}
>
> **Preconditions:** {{required load / data shape / caller behaviour}}
>
> **Suggested remediation:** {{prose only — name the target migration intent, rollback hazard if destructive. No DDL.}}
>
> **Author/executor chain:** `/breakdown` → `schemagen` (per phase) → `/apply` (per phase).
>
> **Scope boundary:**
>   - Do not hand-edit files under `schema/v1/command/**` — `schemagen` authors the migration.
>   - Do not modify the application's Go code in this plan unless a finding explicitly calls for it.
>   - Every destructive step (DROP, ALTER TYPE, NOT NULL backfill, index removal) must carry an
>     explicit rollback step in its phase's exit criteria.
```

## Selection rules for the anchor

Pick the anchor by this precedence:

1. The single highest-severity finding.
2. If multiple findings share that severity, prefer a cohesive group: same taxonomy label, same domain directory (e.g. all under `tables/locations/`), same root cause.
3. Group findings that one migration naturally fixes — e.g. "add FK index" + "change cascade to CASCADE" on the same parent/child pair.
4. If findings are spread across unrelated domains, produce multiple handoff blocks — one per cohesive group. `/breakdown` is better run once per migration unit than once for an unrelated mix.

Never include more than one anchor per block. If the user wants to plan everything at once, produce multiple blocks and let them choose.

## Scope boundaries to include in every handoff

- **DDL authorship belongs to `schemagen`.** The plan's phases invoke `schemagen` as the author step; `/apply` only runs that phase's commands. Never include inline DDL in `/breakdown`'s input.
- **Migration up/down pairs.** Every phase that changes schema must include a rollback migration (part of `schemagen`'s output) — dbaudit surfaces the rollback *hazard*, `schemagen` writes the rollback DDL.
- **Destructive steps are isolated per phase.** A `DROP COLUMN`, `ALTER … TYPE`, or `NOT NULL` backfill is its own phase — never bundled with additive changes, so rollback is single-step.
- **Deploy-order must be preserved.** When adding a new table/function, update `schema/v1/manifest.json` in the same phase that creates the SQL file. `schemagen`'s validation against a scratch DB catches order breakage; dbaudit only points at it.
- **Function rewrites preserve signature when possible.** Changing a function's return type or parameter list breaks every caller; if unavoidable, the plan must include a caller-update phase (which may touch Go code in `lib/`/`service/`/`route/` — out of scope for dbaudit, in scope for the `/breakdown` plan).

## What the handoff must not contain

- DDL (`CREATE`, `ALTER`, `DROP`) — that is `schemagen`'s output, not dbaudit's input.
- Speculative SQL examples or "here's roughly what the migration looks like" prose that reads like DDL.
- Finding ids from earlier reports (unless the user has the report open and references it explicitly).
- Files or symbols not verified in the dbaudit run that produced the handoff.
- Multiple anchors — split into multiple blocks.
