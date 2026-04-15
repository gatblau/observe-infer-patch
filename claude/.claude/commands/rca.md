---
description: Root-cause analysis of an error using inference-mode (ranked hypotheses, no patch).
argument-hint: <error message, stack trace, or log excerpt>
---

# /rca — Root Cause Analysis

You have been given the following error to investigate:

```
$ARGUMENTS
```

If `$ARGUMENTS` is empty, ask the user to paste the exact error message, stack trace, or log excerpt and stop.

Operate strictly in **Inference mode** as defined in `CLAUDE.md` (Working modes → Inference mode). Do **not** propose or apply a patch, even if the cause seems obvious — this command produces analysis only.

## Procedure

1. **Parse the error.** Extract concrete signals: error type/code, message text, file paths, line numbers, function/method names, subjects (NATS), SQL objects, HTTP routes, env vars, timestamps. Treat only what is literally in the input as verified.
2. **Locate evidence in the repo.** Use Grep/Glob/Read to find each concrete signal. Read the exact lines surrounding any matched symbol or call site. Note when a signal cannot be located — that itself is evidence. If the concrete signals suggest artefact drift (missing HTTP route, stale JSON field name, unknown error code string, unresolved env var, DDL vs migration mismatch), note to the user that `/syncheck` may localise the cause faster than bottom-up grepping. Continue the RCA regardless — do not invoke syncheck from here.
3. **If the error touches `schema/**` or any `*.sql`**, layer **SQL & schema safety** rules on top: never name a table/column/function as verified unless visible in the repo.
4. **Generate ranked hypotheses.** Aim for 3–5 plausible root causes, ordered most-to-least likely. For each: one-line claim, the supporting evidence, the contradicting evidence (if any), and what would confirm or refute it.
5. **Do not collapse uncertainty.** If two hypotheses are equally supported, say so. Never present a guessed identifier, signature, or config value as confirmed.

## Required output structure

### Error summary
- One sentence restating the error in your own words, plus the concrete signals you extracted.

### Verified
- Facts directly supported by the input or by repository evidence you read. Cite `path:line` for each repo-derived fact.

### Likely root causes (ranked)
For each hypothesis (most likely first):
- **H{n} — {one-line claim}** *(confidence: high / medium / low)*
  - **Evidence:** what supports it (with `path:line` where applicable)
  - **Against:** what weakens it, or "none observed"
  - **Mechanism:** brief chain from cause → observed error

### What to verify next
- Exact files to read, symbols to grep, log lines to capture, SQL objects to inspect, or commands to run that would discriminate between the top hypotheses. Be specific enough that the user (or a follow-up turn) can act on each item without further interpretation.

### Out of scope
- Note explicitly: no patch is proposed. If the user wants a fix, they should re-invoke in Patch mode after the top hypothesis is verified.