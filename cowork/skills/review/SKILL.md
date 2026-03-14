---
name: review
description: Pre-landing code review. Use when reviewing a PR, checking a diff before merging, or auditing code for production safety. Triggers on "review", "code review", "review this PR", "review before merge", "review this diff", or "is this code safe". Two-pass review — critical issues (SQL safety, race conditions, LLM trust boundaries) and informational issues.
argument-hint: "<PR URL, diff, or file path>"
---

# /review

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../../CONNECTORS.md).

Pre-landing code review. Analyzes the diff against main for structural issues that tests don't catch.

## Usage

```
/review <PR URL, diff, or file path>
```

Review the provided code: @$ARGUMENTS

If no specific file or URL is provided, ask what to review.

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                      /review                                     │
├─────────────────────────────────────────────────────────────────┤
│  STANDALONE (always works)                                      │
│  ✓ Paste a diff or point to files                              │
│  ✓ Pass 1 — Critical: SQL safety, race conditions, trust        │
│  ✓ Pass 2 — Informational: dead code, test gaps, style          │
│  ✓ Terse output: file:line + one-line fix                      │
├─────────────────────────────────────────────────────────────────┤
│  SUPERCHARGED (when GitHub is connected)                        │
│  + Pull PR diff automatically from URL                          │
│  + Read current branch vs main without manual paste             │
└─────────────────────────────────────────────────────────────────┘
```

## Step 1: Check Branch / Get Diff

**If GitHub is connected and a PR URL was provided:**
- Pull the PR diff automatically
- Note the base branch and target branch

**If working standalone:**
- Use the pasted diff or described changes
- Ask for context if needed: "Which branch are these changes on? What was the intent?"

**If on main with no changes:** Output "Nothing to review — you're on main or have no changes against main." and stop.

## Step 2: Two-Pass Review

Apply the checklist below in two passes:
- **Pass 1 (CRITICAL):** SQL & Data Safety, Race Conditions & Concurrency, LLM Output Trust Boundary
- **Pass 2 (INFORMATIONAL):** All remaining categories

### Output format

```
Pre-Landing Review: N issues (X critical, Y informational)

**CRITICAL** (blocking):
- [file:line] Problem description
  Fix: suggested fix

**Issues** (non-blocking):
- [file:line] Problem description
  Fix: suggested fix
```

If no issues found: `Pre-Landing Review: No issues found.`

Be terse. For each issue: one line describing the problem, one line with the fix. No preamble, no summaries, no "looks good overall."

## Review Checklist

### Pass 1 — CRITICAL

#### SQL & Data Safety
- String interpolation in SQL queries (use parameterized queries or ORM methods)
- TOCTOU races: check-then-set patterns that should be atomic (`WHERE` + `update_all`)
- `update_column`/`update_columns` bypassing validations on fields that should have constraints
- N+1 queries: `.includes()` missing for associations used in loops or views

#### Race Conditions & Concurrency
- Read-check-write without uniqueness constraint or race-safe retry
- `find_or_create_by` on columns without unique DB index — concurrent calls can create duplicates
- Status transitions that don't use atomic `WHERE old_status = ? UPDATE SET new_status`
- XSS: `html_safe` on user-controlled data, `raw()`, or string interpolation into HTML output

#### LLM Output Trust Boundary
- LLM-generated values (emails, URLs, names) written to DB or passed to mailers without format validation
- Structured tool output (arrays, hashes) accepted without type/shape checks before database writes

### Pass 2 — INFORMATIONAL

#### Conditional Side Effects
- Code paths that branch on a condition but forget to apply a side effect on one branch
- Log messages that claim an action happened when the action was conditionally skipped

#### Magic Numbers & String Coupling
- Bare numeric literals used in multiple files — should be named constants
- Error message strings used as query filters elsewhere

#### Dead Code & Consistency
- Methods defined but never called
- Duplicate logic that should be extracted

#### LLM Prompt Issues (if applicable)
- Prompt strings that contain user-controlled data without sanitization
- Missing output format validation for structured generation

#### Test Gaps
- Happy paths tested but failure modes missing
- New public methods without any tests
- Tests that stub the exact behavior under test (testing the mock, not the code)

#### View / Frontend
- User-visible strings not in i18n/l10n files
- Hardcoded URLs, colors, or layout values that belong in constants
- Missing loading and error states

### Do NOT flag
- Style preferences that don't affect correctness or safety
- Refactoring opportunities unrelated to the diff
- Anything already addressed elsewhere in the diff

## Step 3: Output Findings

**Always output ALL findings** — both critical and informational.

- **If CRITICAL issues found:** Output all findings, then for EACH critical issue ask separately:
  - The problem (file:line + description)
  - Recommended fix
  - Options: A) Fix it now, B) Acknowledge and merge anyway, C) False positive — skip
  
  After all questions are answered, summarize what the user chose. If the user chose A (fix) on any issue, apply the fixes.

- **If only non-critical issues found:** Output findings. No further action needed.

- **If no issues found:** Output `Pre-Landing Review: No issues found.`

## Important Rules

- **Read the FULL diff before commenting.** Do not flag issues already addressed in the diff.
- **Read-only by default.** Only modify files if the user explicitly chooses "Fix it now" on a critical issue. Never commit, push, or create PRs.
- **Be terse.** One line problem, one line fix. No preamble.
- **Only flag real problems.** Skip anything that's fine.

## If Connectors Available

If **GitHub** is connected:
- Pull the PR diff automatically from a URL
- Check CI status to understand what's already been tested
- Post review comments directly on the PR

If **knowledge base** is connected:
- Check changes against team coding standards and style guides

## Tips

1. **Provide context** — "This handles PII" or "This is in a hot path" helps focus the review.
2. **Specify concerns** — "Focus on security" narrows the review scope.
3. **Include tests** — I'll check test coverage and quality too.
