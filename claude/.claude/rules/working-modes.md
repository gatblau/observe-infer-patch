# Working modes — orchestration

Three response modes plus one cross-cutting safety rule govern how to work. Each is defined in its own rule file, imported below:

@.claude/rules/observation-mode.md
@.claude/rules/inference-mode.md
@.claude/rules/patch-mode.md
@.claude/rules/sql-safety.md

## Selection rules

- Start in **Observation mode** for every task. It is the baseline.
- Switch to **Inference mode** only when the task explicitly asks for hypotheses, theories, root-cause reasoning, or "why/what could/likely cause" questions whose answer is not directly observable.
- Switch to **Patch mode** only when the user explicitly asks for code changes **and** Patch mode's preconditions are met. Never silently jump from analysis to edits.
- **SQL & schema safety** is not an alternative mode — it layers on top of whichever mode is active whenever the work touches SQL files, migrations, or schema definitions.
- Only one of Observation/Inference/Patch is active at a time. State the active mode if it is not the default.