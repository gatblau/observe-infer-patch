---
name: patch-mode
description: Use when the user explicitly wants implementation, file edits, a patch, a diff, or concrete code changes, but only after key files, identifiers, and interfaces are verified.
---

# Patch mode

Use this skill only when the task requires actual code changes.

## Purpose
Turn verified understanding into minimal, controlled edits.

## Preconditions
Before proposing or applying changes, verify:
- target file or files
- exact symbol names
- relevant function, method, or class signatures
- relevant config keys, routes, schema objects, or environment variables
- any external interface affected by the change

If any required detail is not verified:
- stop
- list the missing facts
- do not fill the gaps with plausible code

## Workflow
1. Summarize verified facts
2. List assumptions, if any remain
3. If assumptions remain, stop and surface them clearly
4. Only then propose or apply the smallest viable patch

## Patch rules
- Keep edits minimal and local
- Match existing project conventions
- Do not introduce unverified helper functions
- Do not rename identifiers unless necessary and verified
- Do not change unrelated code for style cleanup
- Explain why each change is needed
- Surface risks and follow-on checks

## Output structure
### Verified
- Facts supporting the change

### Assumptions
- Any unresolved assumptions that block a safe patch

### Proposed patch
- The exact intended change and why

### Risks
- Side effects
- Compatibility concerns
- Tests or validation needed

## Safety
If the repository evidence is incomplete, do not manufacture a complete patch.
