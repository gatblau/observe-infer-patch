# Design ↔ Spec checks

Structural comparison between `docs/design/*.md` and `docs/spec/phase-*.md`. No language assumptions — both sides are Markdown. Use T1 (file existence) and T2 (heading/term extraction) only.

## Aspect 1 — Glossary coverage

**Rule:** Every domain noun introduced in a design document must appear in `docs/spec/phase-1-analysis.md` Glossary (section 1C).

**Extraction (design side):** headings (`##`, `###`) and bolded terms (`**Term**`) that name a domain concept. Exclude generic words (list them in `stopwords` below).

**Extraction (spec side):** the Term column of the Phase 1C Glossary table.

**Finding:**
- Design term has no glossary row → `SCOPE-GAP` (design orphan) or `INTENT-DRIFT` if the design file is newer than Phase 1.
- Glossary row with no design mention → `SCOPE-GAP` (spec orphan); may be legitimate if derived from a Phase 1 assumption.

**Stopwords (default, extend per-project):** system, service, component, user, data, request, response, database, table, error.

## Aspect 2 — Assumption coverage

**Rule:** Every ambiguity a reasonable reader would identify in the design must either:
- appear as a row in Phase 1A Assumptions Register, OR
- appear as a row in Phase 1B Open Questions.

**Extraction (design side):** sentences containing hedging markers — "may", "could", "either", "TBD", "TODO", "?", "not sure", "depends on", "to be decided".

**Extraction (spec side):** rows in Phase 1A and 1B tables.

**Finding:**
- Hedge in design with no matching assumption or open question → `INTENT-DRIFT` (design ambiguity not yet resolved in spec).

This aspect is inherently heuristic. Always tag T2 confidence.

## Aspect 3 — Noun → component mapping

**Rule:** Every significant noun introduced in design (entities, services, subsystems) must map to:
- a row in Phase 2B Component Inventory, OR
- a type in Phase 2C Shared Types Catalogue, OR
- an explicit Phase 1A assumption recording that it is externally owned.

**Extraction (design side):** nouns captured in Aspect 1 that are not already glossary-only concepts.

**Extraction (spec side):** Phase 2B component names, Phase 2C type names.

**Finding:**
- Noun with no mapping → `SCOPE-GAP` (design orphan).
- Phase 2B/2C entry with no design mention → `SCOPE-GAP` (spec orphan).

## Aspect 4 — Action → component method mapping

**Rule:** Every verb-phrase in design that describes a system action ("the system creates …", "users submit …") must map to a Phase 3 component spec's Public Interface section.

**Extraction (design side):** verb phrases in imperative or present-tense sentences. Recognise action markers: "shall", "must", "creates", "emits", "publishes", "subscribes", "returns", "rejects".

**Extraction (spec side):** function/route entries in every Phase 3 component's Public Interface.

**Finding:**
- Action with no Public Interface entry → `INTENT-DRIFT` if design newer; `SCOPE-GAP` if spec predates design.

## Aspect 5 — Cross-cutting mentions

**Rule:** If design explicitly mentions a cross-cutting concern (auth, logging, metrics, rate limiting, pagination, CORS, migrations, health, shutdown), a Phase 4 section must exist for it.

**Extraction (design side):** keyword scan.

**Extraction (spec side):** Phase 4 section headings.

**Finding:**
- Concern mentioned in design but missing Phase 4 section → `INTENT-DRIFT` or `SCOPE-GAP`.

## Aspect 6 — Recency / resolved open questions

**Rule:** Every Phase 1B open question must either be resolved (moved to Phase 1A Assumptions with rationale) or remain blocking.

**Extraction:** Phase 1B rows with `Blocking? = No` that are older than Phase 3 should have been resolved by now.

**Finding:**
- Blocking open question older than Phase 3 → `INTENT-DRIFT` (spec generated past an unresolved blocker).
- Non-blocking question still present after three spec revisions → informational note (not a drift finding).

## What design↔spec checks do not do

- Do not judge whether the design decisions are correct.
- Do not check spelling, prose quality, or formatting.
- Do not compare design to code directly.
