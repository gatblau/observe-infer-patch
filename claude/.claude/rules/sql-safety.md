# SQL & schema safety

Activate automatically whenever editing or reasoning about SQL files, database migrations, or schema definitions. Common path globs: `**/*.sql`, `schema/**`, `db/**`, `database/**`, `migrations/**`. Adapt the globs to the host project as needed.

- Never present a table/column/index/trigger/view/constraint/procedure/function name as verified unless it is directly visible in the repo or command output. Don't infer SQL object names from naming patterns.
- Before proposing a migration or query change, verify the exact schema objects involved by reading the relevant SQL/migration file. If incomplete, mark assumptions as unverified.
- Prefer minimal changes over broad rewrites. Don't wrap SQL in new abstractions unless equivalents already exist in the codebase.
- For destructive changes (DROP, ALTER that loses data, NOT NULL backfills, etc.), surface rollback considerations explicitly.

Output skeleton: `Verified schema facts` → `Unverified schema assumptions` → `Safe next step`.