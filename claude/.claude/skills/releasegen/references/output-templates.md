# Output templates

## Keep a Changelog (default)

```markdown
## [{{next_version}}] — {{YYYY-MM-DD}}

### Breaking changes
- **{{subject}}** ({{scope}}) — {{short-sha}}
  {{migration paragraph}}

### Added
- {{subject}} ({{scope}}) — {{short-sha}}

### Changed
- {{subject}} ({{scope}}) — {{short-sha}}

### Deprecated
- {{subject}} ({{scope}}) — {{short-sha}}

### Removed
- {{subject}} ({{scope}}) — {{short-sha}}

### Fixed
- {{subject}} ({{scope}}) — {{short-sha}}

### Security
- {{subject}} ({{scope}}) — {{short-sha}}

---
Full changelog: {{from}}...{{to}}
```

## Conventional style

Flat list, grouped minimally:

```markdown
# {{next_version}} ({{YYYY-MM-DD}})

## Features
* {{scope}}: {{subject}} ({{short-sha}})

## Bug Fixes
* {{scope}}: {{subject}} ({{short-sha}})

## BREAKING CHANGES
* {{subject}}
  {{migration paragraph}}
```

## Plain style

Single bullet list, no grouping, for simple projects:

```
{{next_version}} ({{YYYY-MM-DD}})
- {{subject}} ({{short-sha}})
- {{subject}} ({{short-sha}})
```

## Release notes narrative

Always produced, regardless of style. Tone differs by audience.

### Internal

```markdown
## Release {{next_version}}

Highlights: {{3–5 sentences, matter-of-fact, highlighting non-trivial changes}}.

Breaking: {{one sentence per breaking change, or "None"}}.

Omitted from the changelog: {{N}} commits (CI, chore, tests).

Validation required before publishing: {{suggestions — e.g. run integration tests, apidiff vs last tag}}.
```

### External

```markdown
## What's new in {{next_version}}

{{Narrative paragraph, user-benefit framing, British English, avoid jargon}}.

If you are upgrading from {{previous_version}}, note: {{breaking-change summary in user-facing terms, or "no breaking changes"}}.
```

## Suggested publish commands

End with this block so the user can act without retyping:

```bash
# Verify clean working tree
git status

# Create annotated tag (message from release notes)
git tag -a {{next_version}} -m "{{next_version}}

{{short release notes}}"

# Push the tag
git push origin {{next_version}}

# Optional: GitHub release
# gh release create {{next_version}} --notes-file <(cat <<'EOF'
# {{release notes}}
# EOF
# )
```

Always include the `git status` check as the first line. Always leave the `gh release` line commented — the user un-comments if they want it.

## Date format

Use `YYYY-MM-DD` in UTC at the moment of generation. Do not guess the release date; use "today" (generation date) with a parenthetical "(draft — update date on actual release)".
