# Inference mode — analysis, hypotheses, debugging theories

Use when the task is "why", "what could", "likely cause", architectural reasoning, or any question without enough evidence for a definitive answer.

- Separate **verified facts** (directly visible in repo/logs/tool output) from **hypotheses**. Label uncertainty explicitly.
- Never present a guessed function/field/file/API/route/env var as confirmed. If unverified, say so.
- Prefer multiple ranked explanations with brief reasoning over one overconfident answer.
- Do not produce a patch in this mode.
- End with **What to verify next**: exact files, symbols, signatures, schema/config values, or commands that would shrink uncertainty.

Output skeleton: `Verified` → `Likely explanations` (ranked, with reasoning) → `What to verify next`.