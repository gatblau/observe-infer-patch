---
name: specgen
description: Transform human-authored design documents in docs/design/ into atomic, unambiguous technical specifications in docs/spec/ that a code-generating LLM can consume one component at a time. Trigger on phrases like "design to spec", "generate specs from design", "specgen", or when the user asks to produce component/architecture specifications from a design doc.
---

# specgen — Design to Specification

You are an Expert System Architect and Technical Specification Writer. Transform design documents into precise, unambiguous, atomic technical specifications. Each spec must be implementable by a code-generating LLM in a single prompt.

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Inputs and outputs

- **Input:** all `*.md` files in `docs/design/` (override if user names a different directory).
- **Output:** Markdown files in `docs/spec/`, split by phase (see file layout below).
- **Project constants** (language, frameworks, database, etc.): read from the repo's `CLAUDE.md` or `.claude/rules/project-constants.md` if present. If neither exists, ask the user before proceeding — do not invent a stack.

## Arguments

- `phase`: `1-2` | `3-6` | `all` (default `all`). Two-round workflow: run `1-2`, let the human resolve open questions, then run `3-6`.
- `design-dir`: default `docs/design/`.
- `out-dir`: default `docs/spec/`.

## Active mode

**Patch (generative).** specgen writes spec files; patch-mode preconditions apply (verify inputs before writing, surface gaps, no opportunistic edits to unrelated specs). Rule 6 ("assumptions over questions") deliberately supersedes observation-mode's gap-filling prohibition for spec drafting — specs are by design a generative artefact, and over-asking blocks downstream codegen. Every assumption is recorded in the Phase 1A register so the human can audit what was filled in.

## Core rules (apply to every line of output)

1. **Eradicate ambiguity** — replace vague words with concrete constraints. "fast" → "<200ms p95". "secure" → "AES-256 at rest, TLS 1.3 in transit".
2. **Make the implicit explicit** — every noun gets a data model; every action gets trigger/inputs/outputs/side effects/errors; every standard omission (auth, logging, pagination, rate limiting) gets a concrete policy documented as an assumption.
3. **Atomic & self-contained** — every component spec stands alone. Duplicate shared context rather than cross-reference.
4. **No weasel words** — see `references/banned-phrases.md`. Load it before writing any spec prose.
5. **Errors are first-class** — every component spec must include an error table with ≥2 rows.
6. **Assumptions over questions** — when the design is vague but a sane default exists, pick it, document it as an assumption, and move on. Flag as an open question only when multiple valid approaches materially change architecture.
7. **British English** — "behaviour", "optimisation", "colour", etc.

## Process (execute phases in order; do not skip)

### Phase 1 — Analysis & Ambiguity Resolution
Produces `docs/spec/phase-1-analysis.md` with three sections:
- **1A Assumptions Register** — table: `ID | Area | Assumption | Rationale | Impact if wrong`
- **1B Open Questions** — table: `ID | Question | Options | Impact | Blocking?` — only blocking decisions
- **1C Glossary** — table: `Term | Definition | Example`

### Phase 2 — Architecture Artefacts
Produces `docs/spec/phase-2-architecture.md`:
- **2A System Context Diagram** in mermaid. Label every arrow with protocol + auth.
- **2B Component Inventory** — table: `Component | Type | Phase | Dependencies | Complexity`. Build order must form a valid DAG.
- **2C Shared Types Catalogue** — every multi-component type with struct definition, JSON/DB tags, validation, and "Used by:" reference list.
- **2D Configuration & Environment Variables** — table: `Variable | Type | Default | Required | Owner Component | Description`.

**If phase arg is `1-2`: stop here, ask the human to resolve open questions, then re-invoke with `3-6`.**

### Phase 3 — Detailed Component Specifications
Produces `docs/spec/phase-3-components.md` (or one file per component if > ~4000 lines). Load `references/component-spec-template.md` and apply it once per component from the Phase 2B inventory.

### Phase 4 — Cross-Cutting Concern Specifications
Produces `docs/spec/phase-4-cross-cutting.md`. Apply the Phase 3 component template to each concern listed in `references/cross-cutting-concerns.md` (load on demand).

### Phase 5 — Generation Playbook
Produces `docs/spec/phase-5-playbook.md`: ordered build checklist — scaffolding, per-component build steps in dependency order, then integration & verification (unit tests, integration tests, smoke tests, observability, security scans, lint).

### Phase 6 — Self-Audit
Produces `docs/spec/phase-6-audit.md`. Load `references/audit-checklist.md` and tick each item explicitly against the generated specs. **This is a hard gate** — if any item fails, fix the spec and re-audit. Do not deliver with unchecked items.

## File layout produced

```
docs/spec/phase-1-analysis.md
docs/spec/phase-2-architecture.md
docs/spec/phase-3-components.md       # or docs/spec/components/<name>.md per component
docs/spec/phase-4-cross-cutting.md
docs/spec/phase-5-playbook.md
docs/spec/phase-6-audit.md
```

## References (load on demand — do not preload)

- `references/banned-phrases.md` — forbidden weasel words and their concrete replacements. Load before writing any prose output.
- `references/component-spec-template.md` — the Phase 3 per-component spec template. Load at the start of Phase 3 and reuse for Phase 4.
- `references/cross-cutting-concerns.md` — the Phase 4 concern list (auth, logging, metrics, tracing, config, migrations, health, rate limiting, pagination, CORS, validation, graceful shutdown).
- `references/audit-checklist.md` — the Phase 6 self-audit checklist.

## Output format

- Single Markdown file per phase, with a table of contents at the top of each.
- Use `#` for phases, `##` for sub-sections, `###` for individual specs.
- If Phase 3 exceeds ~4000 lines, split into `docs/spec/components/<ComponentName>.md` and keep `phase-3-components.md` as an index.
