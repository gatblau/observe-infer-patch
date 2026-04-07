---
paths:
  - "db/**"
  - "database/**"
  - "migrations/**"
  - "**/*.sql"
  - "**/schema/**"
---

# SQL and schema safety

## Scope
Apply these rules whenever working with database files, migrations, schema definitions, or SQL.

## Safety rules
- Never present a table, column, index, trigger, view, constraint, procedure, or function name as verified unless it is directly visible in the repository or command output.
- Never infer SQL object names from naming patterns and present them as facts.
- Before proposing a migration or query change, verify the exact schema objects involved.
- If the schema is incomplete or not visible, explicitly mark the missing details as unverified.

## Patch behavior
- Prefer minimal changes over broad rewrites.
- Avoid "helpful" generated abstractions around SQL unless they already exist in the codebase.
- Surface rollback considerations for destructive changes.

## Output structure
### Verified schema facts
### Unverified schema assumptions
### Safe next step
