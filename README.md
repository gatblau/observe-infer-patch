# Observe, Infer, Patch Code Agent Pattern

This repository defines a strict 3-phase workflow for coding agents:

1. **Observe** → Root Cause Analysis
2. **Infer** → Fix Plan
3. **Patch** → Minimal Implementation

The goal is to make coding agents behave like a disciplined engineering assistant that:

- separates evidence from reasoning,
- avoids fabricated identifiers and APIs,
- does not jump directly from symptoms to code changes,
- states uncertainty explicitly,
- patches only after the plan is grounded.

---

## Why this exists

A common failure mode in LLM-assisted coding is:

1. seeing partial evidence,
2. inventing a plausible cause,
3. inventing a plausible function or API,
4. producing a confident but incorrect patch.

This repo enforces a safer pattern:

- **Observe**: identify the most likely root cause using only what is shown
- **Infer**: create a minimal remediation plan
- **Patch**: implement only the confirmed plan using verified identifiers

