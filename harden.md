---
name: harden
description: "Run a harden loop on code: simplify → lint → test → review → repeat until clean."
argument-hint: "[file1 file2 ...] — files to harden (defaults to git-changed files)"
---

# Harden Loop

Iteratively improve code quality by cycling through simplification, linting, testing, and review until the code reviewer finds no issues.

## Inputs

- `$ARGUMENTS` — Optional list of files to harden. If omitted, auto-detect from `git diff --name-only origin/main`.

## Step 0 — Identify Target Files

If no files were provided, detect them:

```bash
git diff --name-only origin/main
```

Filter to only code files (`.php`, `.ts`, `.tsx`, `.js`, `.jsx`). Ignore test files for the simplifier pass — but include them for the test step.

If no changed files are found, stop and tell the user there's nothing to harden.

Show the file list to the user before starting.

## The Loop

Run autonomously. No checkpoints inside the loop. Report results at the end.

Track the current round number starting at 1. Cap at **5 rounds** to avoid infinite loops — if still failing after 5 rounds, stop and report remaining issues.

### Step 1 — Code Simplifier

Launch the `code-simplifier` subagent on the target files.

Tell it which files to review and provide context on what the code does.

If it makes changes, note them for the next step.

### Step 2 — Lint

Run your project linter on the changed files. For PHP/Laravel projects using Pint:

```bash
php vendor/bin/pint
```

Adapt this command to your project's linter (`eslint`, `phpcs`, `rubocop`, etc.).

If the linter makes changes, that's fine — they'll be reviewed in Step 4.

### Step 3 — Run Tests

Look for test files related to the changed files. Common patterns:

- `tests/Unit/` or `tests/Feature/` mirrors of the source path
- Files ending in `Test.php`, `.test.ts`, `.spec.ts`

**Safety check**: Read the test file before running. If it touches a real database without a rollback/transaction guard, STOP and warn the user. Do not run it.

If tests exist, run them:

```bash
php artisan test --filter={TestName}
```

Adapt to your test runner (`jest`, `pytest`, `go test`, etc.).

If no test files are found, skip this step and note it in the summary.

If tests fail, stop the loop immediately. Report the failure. Do not continue.

### Step 4 — Code Reviewer

Launch the `code-reviewer` subagent on all target files.

Provide it with:
- The list of files
- What round this is
- What changes were made in Steps 1–2 of this round
- Instructions to do a thorough review

The reviewer will classify issues as: **Critical**, **Warning**, or **Suggestion**.

### Step 5 — Evaluate and Loop

Read the reviewer's output.

- **If Critical or Warning issues found**: Fix them, then go back to Step 2 (Lint) and continue the loop. Increment the round number.
- **If only Suggestions or no issues**: The loop is done. Proceed to the summary.

Suggestions are informational — do not fix them automatically. Include them in the summary for the user to decide.

## Summary

When the loop completes (either clean or max rounds reached), report:

```
## Harden Results

**Rounds**: {n}
**Status**: {Clean ✓ | Stopped — max rounds | Stopped — test failure}

### Changes Made
- Round 1: {what changed}
- Round 2: {what changed}

### Remaining Suggestions (if any)
- {suggestion from reviewer}

### Files Touched
- `path/to/file1`
- `path/to/file2`
```

## Important Notes

- **Lint after any code change**: Any time code is modified (by simplifier, reviewer fixes, or manual edits), run lint before the next review.
- **Max 5 rounds**: Safety valve. If the reviewer keeps finding issues after 5 rounds, something deeper is wrong — report and let the user decide.
- **Tests are a hard stop**: If tests fail, do not continue. The code must pass tests before more changes.
- **Suggestions are optional**: Only fix Critical and Warning issues automatically. Surface Suggestions to the user.
