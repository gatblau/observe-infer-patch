# Grouping and filtering

## Keep a Changelog buckets

Standard order, always this sequence:

1. Added
2. Changed
3. Deprecated
4. Removed
5. Fixed
6. Security

Omit an empty bucket entirely rather than printing "None".

## Line format per commit

```
- <subject> (<scope>) — <short-sha>
```

- Strip the conventional prefix (`feat:`, `fix:`) from the subject when emitting the line; the bucket already conveys the type.
- Keep the scope in parentheses if present.
- Capitalise first letter of subject.
- Truncate subjects at ~100 chars with an ellipsis; consumers can click through to the commit.
- For breaking commits, prefix the line with `**BREAKING:** `.

## Filtering rules

Drop from the user-facing changelog (but count):
- `chore:`, `ci:`, `build:`, `test:`, `style:` — noise for consumers.
- `docs:` — keep only for `audience: internal`.
- Merge commits (`Merge branch`, `Merge pull request #123`) when they don't add novel content.
- Revert pairs: if commit A is reverted by commit B in the same range, drop both and note "reverted" only if no net change.
- Bot commits (dependabot / renovate) — bucket as "Dependencies updated (N commits)" one-liner instead of listing every bump.

## Dedup / merge

When multiple commits describe the same feature (e.g. `feat: add X`, `fix: fix X typo`, `test: tests for X`), merge them into one line in the higher-severity bucket. Keep the feat line; drop the fix and test.

This requires judgement — be conservative. Only merge when the subject clearly references the same feature.

## Breaking-change callout

In addition to the inline `**BREAKING:**` prefix within the normal bucket, emit a separate top-level **Breaking changes** section above the standard buckets, listing each breaking commit with its migration paragraph. Consumers skim this first.

## Authors section (optional)

For `audience: external`, append a "Thanks" section listing unique commit authors. Exclude bots.

For `audience: internal`, omit.

## Commit range footer

End with:
```
Full changelog: <from>...<to>
```
Use the project's remote URL if detectable to produce a clickable compare link:
```
Full changelog: https://github.com/org/repo/compare/v1.4.2...v1.5.0
```
