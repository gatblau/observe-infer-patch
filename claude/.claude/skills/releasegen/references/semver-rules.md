# Semver bump rules

## Standard rules

Given the highest-severity change in the range:

| Highest change | Bump |
|---|---|
| Any breaking | major |
| Any `feat` (no breaking) | minor |
| Only `fix` / `perf` | patch |
| Only `chore` / `docs` / `ci` / `test` / `build` | patch (publish optional — sometimes skipped) |

## Pre-1.0 convention

Projects at `0.y.z` may follow the convention: breaking changes bump `y` (minor), features bump `z` (patch). Detect by:
- Current version starts with `0.`.
- README contains "unstable" / "pre-1.0" / "no stability guarantees".

If pre-1.0 convention applies, map: breaking → minor, feature → patch, fix → patch. State the convention in the bump justification.

## First release

If no tag exists:
- Default to `0.1.0`.
- Recommend `1.0.0` if README / docs signal production use or stability commitment.

## Override handling

If the user passed `bump`:
- Honour it.
- Compare against the calculated recommendation.
- If they conflict, emit a warning: e.g. "User requested patch, but N breaking changes are present. Proceeding with patch as requested — acknowledge this is not SemVer-compliant."

## Source of current version

In order of preference:
1. Most recent git tag matching `v?\d+\.\d+\.\d+` reachable from HEAD.
2. Version field in `package.json`, `pyproject.toml`, `Cargo.toml`.
3. A `VERSION` file at repo root.
4. ldflags injection hint in `Makefile` (e.g. this project's `lib.Version`).

If multiple sources disagree, state all values and use the tag as authoritative.

## Output format

```
Current: v1.4.2
Next:    v2.0.0
Bump:    major
Reason:  1 breaking change (commit abc1234 — 'feat!: remove /api/v1/legacy')
```

If no bump is warranted (docs/chore only), state:
```
No version bump recommended — no user-visible changes.
Consider publishing anyway if CI artefacts / docs should be versioned.
```
