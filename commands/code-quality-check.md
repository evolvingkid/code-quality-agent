---
description: Review the current code changes for quality issues — naming, complexity, duplication, dead code, error handling, and tests.
argument-hint: "[path] — optional file or dir to focus on (defaults to the git diff)"
allowed-tools: Read, Grep, Glob, Bash(git diff:*), Bash(git status:*), Bash(git log:*)
---

# Code Quality Check

Determine the scope to review:
- If `$ARGUMENTS` is provided, review that file or directory.
- Otherwise, review the current change set: run `git diff HEAD` (fall back to
  `git diff` / `git status` if needed). If there is no diff, review the most
  recently modified source files.

Then evaluate the code against this checklist and report findings:

1. **Naming** — are names clear, consistent, and intention-revealing?
2. **Complexity** — overly long functions, deep nesting, unclear control flow.
3. **Duplication** — repeated logic that should be extracted or reused.
4. **Dead code** — unused variables, imports, functions, unreachable branches.
5. **Error handling** — swallowed errors, missing edge cases, unsafe assumptions.
6. **Tests** — is new/changed logic covered? Obvious gaps?

## Output format

Group findings by severity: **Blocking**, **Recommended**, **Nitpick**.
For each finding give `file:line`, a one-line description, and a concrete fix.
End with a 1–2 sentence overall assessment.

Do NOT edit files — this is a review. Only suggest changes unless the user
explicitly asks you to apply them.
