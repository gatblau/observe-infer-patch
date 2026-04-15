# Command injection

## Target patterns

Any invocation of a shell or subprocess where an argument is constructed from a variable.

### Go
- `exec.Command("sh", "-c", userInput)` — shell-eval form is the red flag.
- `exec.Command(binary, args...)` with `args` built via `strings.Split(userInput, " ")`.

### Python
- `subprocess.Popen(cmd, shell=True)` with string construction.
- `os.system(...)` with anything other than a constant literal.

### Node.js
- `child_process.exec(cmd)` (shell) vs `execFile(bin, args)` (no shell).
- `spawn(cmd, {shell: true})`.

## Severity

- **Critical:** `shell=True` / `sh -c` form with user input, no validation.
- **High:** argv form (`execFile`, `exec.Command` without `sh -c`) with user-controlled binary path.
- **Medium:** argv form with user-controlled flag value on a known binary (leakage via flag injection possible).
- **Low:** shell invocation with constant input (defence-in-depth — switch to argv form).

## Safe patterns

- argv form (`exec.Command(bin, arg1, arg2)`) with a constant binary and validated arguments.
- Wrapping arguments in a strict allowlist before passing.
- Using language-native APIs instead of shelling out (e.g. `os.Rename` over `mv`).

## Evidence

Show the call, the source of the user-controlled variable, and any allowlist/escape between them.
