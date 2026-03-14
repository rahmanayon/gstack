---
name: ship
description: Fully automated ship workflow. Use when ready to land a feature branch — syncs main, runs tests, bumps version, updates changelog, commits, pushes, and opens a PR. Triggers on "ship", "ship this", "land this branch", "create PR", "push and PR". Non-interactive — just runs. Do NOT use when still deciding what to build.
argument-hint: "[optional: branch name or PR title]"
---

# /ship

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../../CONNECTORS.md).

You are running the `/ship` workflow. This is a **non-interactive, fully automated** workflow. Do NOT ask for confirmation at any step. The user said `/ship` which means DO IT. Run straight through and output the PR URL at the end.

## Usage

```
/ship
```

Run from the feature branch you want to ship.

## Only stop for

- On `main` branch (abort)
- Merge conflicts that can't be auto-resolved (stop, show conflicts)
- Test failures (stop, show failures)
- Pre-landing review finds CRITICAL issues and user chooses to fix
- Greptile VALID issues requiring user decision
- MINOR or MAJOR version bump needed (ask once — see Step 4)

## Never stop for

- Uncommitted changes (always include them)
- Version bump choice for MICRO or PATCH (auto-pick)
- CHANGELOG content (auto-generate from diff)
- Commit message approval (auto-commit)

---

## Step 1: Pre-flight

1. Check the current branch. If on `main`, **abort**: "You're on main. Ship from a feature branch."
2. Run `git status` — uncommitted changes are always included.
3. Run `git diff main...HEAD --stat` and `git log main..HEAD --oneline` to understand what's being shipped.

---

## Step 2: Merge origin/main (BEFORE tests)

Fetch and merge `origin/main` so tests run against the merged state:

```bash
git fetch origin main && git merge origin/main --no-edit
```

**If there are merge conflicts:** Try to auto-resolve simple conflicts (VERSION, CHANGELOG ordering). If conflicts are complex or ambiguous, **STOP** and show them.

**If already up to date:** Continue silently.

---

## Step 3: Run tests

Run the test suite. Look for any of these patterns in the project:

```bash
# Try these in order, run whichever exists:
npm test 2>&1 | tee /tmp/ship_tests.txt &
yarn test 2>&1 | tee /tmp/ship_tests.txt &
bundle exec rspec 2>&1 | tee /tmp/ship_tests.txt &
python -m pytest 2>&1 | tee /tmp/ship_tests.txt &
go test ./... 2>&1 | tee /tmp/ship_tests.txt &
bun test 2>&1 | tee /tmp/ship_tests.txt &
wait
```

**If any test fails:** Show the failures and **STOP**. Do not proceed.

**If all pass:** Continue silently — just note the counts briefly.

---

## Step 3.5: Pre-Landing Review

Review the diff for structural issues that tests don't catch.

1. Run `git diff origin/main` to get the full diff.
2. Apply the review checklist in two passes:
   - **Pass 1 (CRITICAL):** SQL & Data Safety, LLM Output Trust Boundary, Race Conditions
   - **Pass 2 (INFORMATIONAL):** All remaining categories (see `/review` skill)
3. **Always output ALL findings.**
4. Output a summary header: `Pre-Landing Review: N issues (X critical, Y informational)`
5. **If CRITICAL issues found:** For EACH critical issue, ask separately with problem description, recommended fix, and options: A) Fix it now, B) Acknowledge and ship anyway, C) False positive — skip.
   - If user chose A on any issue: apply fixes, commit only the fixed files, **STOP** and tell user to run `/ship` again.
   - If user chose only B or C: continue.
6. **If no critical issues:** Output findings and continue.

---

## Step 3.75: Greptile Review (if GitHub connected)

**If GitHub is not connected or no PR exists yet:** Skip this step silently and continue to Step 4.

**If GitHub is connected and a PR exists:**

1. **Fetch Greptile comments:** Using the GitHub connection, read both line-level review comments and top-level comments on the PR, filtering for comments posted by `greptile-apps[bot]`.

   **If no Greptile comments found:** Note `Greptile: No comments.` and continue to Step 4.

2. **Check suppressions:** Read `~/.gstack/greptile-history.md` if it exists. Each line has the format:
   ```
   <YYYY-MM-DD> | <owner/repo> | <type> | <file-pattern> | <category>
   ```
   Skip any comment matching a known false positive (`type == fp`) for the current repo and file pattern.

3. **Classify each non-suppressed comment** (read file at path:line ±10 lines, cross-reference against full diff):
   - **VALID** — a real bug or correctness problem in the current code
   - **ALREADY-FIXED** — a real issue addressed in a subsequent commit on the branch; identify the fixing commit SHA
   - **FALSE-POSITIVE** — misunderstands the code or is stylistic noise

4. Output: `+ N Greptile comments (X valid, Y fixed, Z FP)`

5. **ALREADY-FIXED comments:** Reply automatically using the GitHub connection:
   - Reply text: `"Good catch — already fixed in <commit-sha>."`
   - Save to `~/.gstack/greptile-history.md` (type: `already-fixed`). Ensure `mkdir -p ~/.gstack` first.

6. **FALSE-POSITIVE comments:** Reply automatically using the GitHub connection:
   - Reply text: a concise explanation of why the comment is incorrect
   - Save to `~/.gstack/greptile-history.md` (type: `fp`)

7. **VALID comments:** Ask the user for each one:
   - The comment: file:line (or `[top-level]`) + description + permalink
   - Your recommended fix
   - Options: A) Fix it now, B) Acknowledge and ship anyway, C) It's a false positive
   - If user chooses A: apply the fix, commit the fixed files (`git add <fixed-files> && git commit -m "fix: address Greptile review — <brief description>"`), reply `"Fixed in <commit-sha>."`, save to history (type: `fix`).
   - If user chooses C: reply explaining the false positive, save to history (type: `fp`).

8. **If any VALID fixes were applied:** Re-run tests (Step 3) before continuing to Step 4.

Save Greptile results — they go into the PR body in Step 8.

(Categories for history file: `race-condition`, `null-check`, `error-handling`, `style`, `type-safety`, `security`, `performance`, `correctness`, `other`)

---

## Step 4: Version bump (auto-decide)

1. Read the current `VERSION` file (4-digit format: `MAJOR.MINOR.PATCH.MICRO`, or semver `MAJOR.MINOR.PATCH`)
2. **Auto-decide the bump level:**
   - Count lines changed: `git diff origin/main...HEAD --stat | tail -1`
   - **MICRO** (4th digit, 4-digit versioning) or **PATCH** (semver): < 50 lines changed, trivial tweaks, typos, config
   - **PATCH** (4-digit) or **MINOR** (semver): 50+ lines changed, bug fixes, small-medium features
   - **MINOR** (4-digit) or **MAJOR** (semver): **ASK the user** — only for major features or significant architectural changes
   - **MAJOR:** **ASK the user** — only for milestones or breaking changes
3. Compute the new version (bumping a digit resets all digits to its right to 0).
4. Write the new version to the `VERSION` file.

---

## Step 5: CHANGELOG (auto-generate)

1. Read `CHANGELOG.md` to understand the format.
2. Auto-generate the entry from ALL commits on the branch:
   - `git log main..HEAD --oneline` — every commit being shipped
   - `git diff main...HEAD` — the full diff
   - Categorize: `### Added`, `### Changed`, `### Fixed`, `### Removed`
   - Write concise, descriptive bullet points
   - Insert after the file header, dated today
   - Format: `## [X.Y.Z] - YYYY-MM-DD`
3. **Do NOT ask the user to describe changes.** Infer from the diff and commit history.

---

## Step 6: Commit (bisectable chunks)

**Goal:** Small, logical commits that work well with `git bisect`.

1. Analyze the diff and group changes into logical commits. Each commit = one coherent change.
2. **Commit ordering:**
   - Infrastructure: migrations, config, route additions
   - Models & services (with their tests)
   - Controllers & views (with their tests)
   - VERSION + CHANGELOG: always last
3. **Rules:**
   - A model and its test file go in the same commit
   - A service and its test go in the same commit
   - If the total diff is small (< 50 lines, < 4 files), a single commit is fine
4. Each commit must be independently valid — no broken imports.
5. Final commit format:

```bash
git commit -m "chore: bump version and changelog (vX.Y.Z)

Co-Authored-By: Claude <noreply@anthropic.com>"
```

---

## Step 7: Push

```bash
git push -u origin <branch-name>
```

---

## Step 8: Create PR

**If GitHub is connected:**

Use GitHub to create a pull request:
- Title: `<type>: <summary>` (type = feat/fix/chore/refactor/docs)
- Body template:

```markdown
## Summary
<bullet points from CHANGELOG>

## Pre-Landing Review
<findings from Step 3.5, or "No issues found.">

## Greptile Review
<If Greptile comments were found in Step 3.75: bullet list with [FIXED] / [FALSE POSITIVE] / [ALREADY-FIXED] / [VALID] tag + one-line summary per comment>
<If no Greptile comments found: "No Greptile comments.">
<If GitHub was not connected during Step 3.75: omit this section entirely>

## Test plan
- [x] All tests pass (N total, 0 failures)

🤖 Generated with [Claude Code](https://claude.ai)
```

**If GitHub is not connected:**

Run locally:
```bash
gh pr create --title "<type>: <summary>" --body "..."
```

**Output the PR URL** — this should be the final output the user sees.

---

## Important Rules

- **Never skip tests.** If tests fail, stop.
- **Never force push.** Use regular `git push` only.
- **Never ask for confirmation** except for MINOR/MAJOR version bumps and CRITICAL review findings.
- **The goal is:** user says `/ship`, next thing they see is the review summary + PR URL.

## If Connectors Available

If **GitHub** is connected:
- Create the PR using the GitHub API (no need for `gh` CLI)
- Post comments on the PR with review findings

If **project tracker** (Linear/Jira) is connected:
- Link the PR to the related issue/ticket
