---
name: releasegen
description: Turn a commit range into a changelog entry, a semver version bump recommendation, and release notes. Groups commits by conventional-commit type (or falls back to heuristic classification), highlights breaking changes, and drafts release notes suitable for a tag description or GitHub release. Trigger on phrases like "releasegen", "cut a release", "generate release notes", "changelog for last X commits", "what's changed since <tag>", or when the user asks to prepare a version bump. Do NOT trigger to actually create tags, push, or publish — this skill only drafts text.
---

# releasegen — Release notes and version bump draft

Turn a commit range into release-ready text: a version bump recommendation, a grouped changelog fragment, and longer-form release notes. Does not tag, push, or publish — drafting only.

This skill ships with the repo under `.claude/skills/`. Repo conventions in `CLAUDE.md` and `.claude/rules/*.md` apply — the skill operates within them, not around them.

## Arguments

- `from`: baseline ref. Default: the most recent tag reachable from HEAD (`git describe --tags --abbrev=0`). Accepts any git ref.
- `to`: target ref. Default: `HEAD`.
- `bump`: override the semver bump recommendation. One of `major`, `minor`, `patch`, `auto` (default).
- `style`: output style. `keepachangelog` (default — Keep a Changelog format) | `conventional` | `plain`.
- `audience`: `internal` (default — technical team) | `external` (end-user release notes, less jargon).

## Active mode

**Observation.** Drafts text only — no tags, no pushes, no `gh release`, no edits to `CHANGELOG.md`/`package.json`/`go.mod`. Suggested publish commands are emitted as a fenced block for the user to run themselves.

## Hard rules

- **Drafting only.** No `git tag`, `git push`, no `gh release create`. End with the drafted text and a suggested command the user can run themselves.
- **Bump recommendation must be justified.** State the reason — "breaking change present" / "only additive features" / "fixes only". If `apidiff` was run earlier in the session, reference its findings.
- **Do not invent commits.** Every line in the changelog traces to a real commit in the range. Omit commits that don't belong in user-facing notes (CI tweaks, typos, test-only changes) but state the omit count.
- **British English** for narrative sections. Technical terms in commits stay as-authored.

## Process

### Phase 1 — Resolve range
- Determine `from` (default: `git describe --tags --abbrev=0` — if no tags, use the initial commit and state "first release").
- Determine `to` (default: `HEAD`).
- Capture the list: `git log --pretty=format:"%H%x09%s%x09%an%x09%ae" <from>..<to>`.
- Also capture body for breaking-change detection: `git log --pretty=format:"%H%n%B%n---END---" <from>..<to>`.

### Phase 2 — Classify each commit
Load `references/commit-classification.md`. For each commit, attempt classification in this order:

1. **Conventional commit prefix** (`feat:`, `fix:`, `chore:`, `docs:`, `refactor:`, `test:`, `perf:`, `build:`, `ci:`, `revert:`) with optional scope `feat(scope):` and breaking marker `feat!:` or `BREAKING CHANGE:` in body.
2. **Heuristic** from subject: verbs like "add"/"implement" → feature; "fix"/"resolve" → fix; "update"/"bump" → chore; "refactor" → refactor; etc.
3. **Fallback** "other" — surface these for manual review rather than guessing.

Record per commit: SHA, type, scope, subject, breaking (yes/no), author.

### Phase 3 — Bump recommendation
Load `references/semver-rules.md`. Determine:

- Any breaking commit → **major**.
- Any feature (`feat`) and no breaking → **minor**.
- Only fixes / chores / docs → **patch**.
- If first release: propose `0.1.0` or `1.0.0` depending on stability signals (presence of `stability:` in README, explicit v1 intent).

If `bump` was explicitly passed, use it but warn if it conflicts with the classification (e.g. user asked `patch` but there's a breaking commit).

Record current version from the most recent tag (strip a leading `v`). Emit next version = current + bump.

### Phase 4 — Group and filter
Load `references/grouping.md`. Group commits into Keep a Changelog buckets:

- **Added** — `feat`.
- **Changed** — `refactor`, `perf`.
- **Deprecated** — commits with `deprecated` in subject or `DEPRECATED:` in body.
- **Removed** — commits with `remove`/`drop` as primary verb and no `fix` classification.
- **Fixed** — `fix`.
- **Security** — commits with scope `security` or subject containing `CVE`, `vulnerability`, `secscan`.

Filter out from user-facing notes:
- `chore`, `ci`, `build`, `test`, `docs` (unless `audience: internal`, where `docs` and `build` may be retained).
- Merge commits (`Merge branch`, `Merge pull request`) when the merged work is already represented by its constituent commits.
- Revert pairs (A followed by `Revert A`) — remove both, note as "reverted" if the revert is the head commit.

Always state the filter counts: "Omitted N commits (CI / chore / tests)".

### Phase 5 — Assemble output
Load `references/output-templates.md`. Produce:

1. **Version bump line** — `vX.Y.Z → vA.B.C (major|minor|patch, reason: ...)`.
2. **Changelog fragment** — in the requested style.
3. **Release notes** — narrative paragraph highlighting the 3–5 most important changes, breaking changes prominent at the top. Tone per `audience`.
4. **Breaking-change migration notes** — for each breaking commit, one paragraph telling consumers what to change. If apidiff output is referenced, pull its migration notes verbatim.
5. **Suggested publish commands** — the exact commands the user would run to tag and publish. Do NOT execute them.

### Phase 6 — Present
Output all five sections. End with the explicit note that nothing was written or tagged.

## Output files

Does not write files. If the user asks releasegen to update `CHANGELOG.md`, that's a follow-up `/apply` invocation. Offer the fragment as text.

## References (load on demand)

- `references/commit-classification.md`
- `references/semver-rules.md`
- `references/grouping.md`
- `references/output-templates.md`

## What releasegen does not do

- Does not tag, push, or publish.
- Does not modify `CHANGELOG.md`, `package.json`, `go.mod`, or any version file. Those edits go through `/apply` after the user approves the draft.
- Does not generate release notes for a range that has not been merged (draft PRs). If asked, narrow the range to merged commits.
- Does not run tests or build artefacts. Assume the user gates release on their own CI.
