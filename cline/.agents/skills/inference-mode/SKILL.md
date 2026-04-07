---
name: inference-mode
description: Use for root-cause analysis, possible explanations, likely causes, debugging hypotheses, and architectural reasoning when the user wants theories or ranked possibilities without treating them as verified facts.
---

# Inference mode

Use this skill only when the task requires analysis beyond direct observation.

## Purpose
Generate useful hypotheses without collapsing uncertainty into false certainty.

## Rules
- Separate verified facts from hypotheses.
- Label uncertainty explicitly.
- Do not present guessed identifiers as confirmed.
- If a function, field, file, or API is not verified, call it possible or unverified.
- Prefer multiple plausible explanations over one overconfident answer when evidence is incomplete.
- Rank likely explanations when appropriate.

## Output structure
### Verified
- Facts directly supported by repository evidence, logs, visible files, or command output

### Likely explanations
- Ranked hypotheses
- Short reasoning for each hypothesis
- Clear note where evidence is missing

### What to verify next
- Exact files
- Exact symbols
- Exact signatures
- Exact schema or config values
- Exact commands or searches that would reduce uncertainty

## Boundaries
- Do not generate a code patch by default.
- Do not recommend edits that depend on unverified identifiers.
- Do not rewrite observed facts to fit a hypothesis.

## Quality bar
A good answer in this mode is useful, explicit about uncertainty, and easy to falsify.
