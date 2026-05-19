# Harden Loop

A Claude Code skill that iteratively improves code quality by cycling through simplification, linting, testing, and review until the code reviewer finds no issues.

## What It Does

Runs a multi-agent loop on your changed files:

```
Simplify → Lint → Test → Review → repeat until clean
```

Each round, a code-simplifier agent cleans up the code, Pint fixes style issues, tests verify nothing broke, and a code-reviewer agent checks for Critical and Warning issues. If issues are found, the loop repeats. If only Suggestions remain, the loop exits and surfaces them for your review.

## Requirements

- [Claude Code](https://claude.ai/code) CLI
- PHP project using [Laravel Pint](https://laravel.com/docs/pint) for linting
- PHPUnit or Pest for tests
- Claude Code custom agents: `code-simplifier` and `code-reviewer` (see below)

## Installation

Copy `harden.md` into your Claude Code commands directory:

```bash
# Project-level (recommended)
cp harden.md .claude/commands/harden.md

# Or globally
cp harden.md ~/.claude/commands/harden.md
```

Then run it with:
```
/harden
```

Or target specific files:
```
/harden src/Services/PaymentService.php src/Actions/ChargeCard.php
```

## Custom Agents

The skill relies on two Claude Code custom agents. Add these to your `.claude/agents/` or `~/.claude/agents/`:

**`code-simplifier.md`**: Reviews code for over-engineering, duplication, and unnecessary complexity. Makes it simpler while preserving behavior.

**`code-reviewer.md`**: Reviews code for correctness, security, and maintainability. Classifies issues as Critical, Warning, or Suggestion.

You can use any agent definitions you have; the skill just needs agents with those names.

## Customizing for Your Project

The skill auto-detects changed files from `git diff --name-only origin/main`. Edit `harden.md` to change:

- **Branch**: replace `origin/main` with your default branch (`origin/master`, etc.)
- **Lint command**: replace the Pint command with your linter (`eslint`, `phpcs`, etc.)
- **Test command**: replace with your test runner (`phpunit`, `jest`, `pytest`, etc.)
- **File extensions**: adjust the filter to match your stack

## How the Loop Works

1. **Detect files**: changed files from git, filtered to code files only
2. **Simplify**: code-simplifier agent reviews and refines
3. **Lint**: Pint fixes style issues automatically
4. **Test**: runs related test files; hard-stops if tests fail
5. **Review**: code-reviewer classifies issues: Critical / Warning / Suggestion
6. **Evaluate**: if Critical or Warning issues found, loop repeats from step 2; if clean, exit

Maximum 5 rounds. If still failing after round 5, the loop stops and reports remaining issues.

## Example Output

```
## Harden Results

**Rounds**: 2
**Status**: Clean ✓

### Changes Made
- Round 1: Simplified nested conditionals in PaymentService; extracted magic numbers to constants
- Round 2: No simplifier changes; Pint fixed trailing comma style

### Remaining Suggestions
- Consider extracting charge logic to a dedicated ChargeAction for testability

### Files Touched
- `src/Services/PaymentService.php`
- `src/Actions/ChargeCard.php`
```

## License

MIT
