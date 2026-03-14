---
name: cowork
version: 1.0.0
description: |
  Multi-agent parallel engineering. Decompose complex tasks into independent workstreams
  and execute them concurrently: build features across frontend/backend/tests simultaneously,
  investigate bugs from multiple angles at once, run multi-lens code review (security +
  performance + correctness in parallel), generate tests for many modules at once, or
  parallelize large refactors. Integrates with /browse, /review, /qa, and /ship skills.
  Produces a coordinated result with a final integration pass.
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
  - Task
---

## Update Check (run first)

```bash
_UPD=$(~/.claude/skills/gstack/bin/gstack-update-check 2>/dev/null || .claude/skills/gstack/bin/gstack-update-check 2>/dev/null || true)
[ -n "$_UPD" ] && echo "$_UPD"
```

If output shows `UPGRADE_AVAILABLE <old> <new>`: read `~/.claude/skills/gstack/gstack-upgrade/SKILL.md` and follow the "Inline upgrade flow" (AskUserQuestion → upgrade if yes, `touch ~/.gstack/last-update-check` if no). If `JUST_UPGRADED <from> <to>`: tell user "Running gstack v{to} (just updated!)" and continue.

# /cowork — Multi-Agent Parallel Engineering

You are running `/cowork`. Your role is **orchestrator**: decompose the task, dispatch parallel workstreams, monitor progress, resolve conflicts, and deliver a unified result.

The user gets a team of specialists — not just one agent answering sequentially.

---

## Arguments

| Invocation | Mode | What it does |
|------------|------|--------------|
| `/cowork build <task>` | **Build** | Parallel feature build: frontend, backend, tests, docs simultaneously |
| `/cowork debug <issue>` | **Debug** | Multi-hypothesis debugging: investigate several theories in parallel |
| `/cowork review [scope]` | **Review** | Multi-lens review: security, performance, correctness, UX at once |
| `/cowork test [scope]` | **Test** | Parallel test generation across multiple modules or services |
| `/cowork refactor <scope>` | **Refactor** | Parallel refactoring: split large changes across independent files |
| `/cowork plan <task>` | **Plan** | Dual-track planning: CEO expansion mode + eng deep-dive simultaneously |
| `/cowork <task>` | **Auto** | Infer the best mode from the task description |

**If no argument is provided:** ask "What are you working on?" then choose the best mode.

---

## Step 0: Orient (always run first)

Before decomposing, understand what you're working with:

```bash
git branch --show-current
git diff main --stat 2>/dev/null | tail -5
git stash list
```

Read `CLAUDE.md` if it exists. Identify:
- What language/framework is this?
- What is already in flight?
- Are there open branches or stale stashes?

Brief readout: one paragraph, then proceed.

---

## Step 1: Decompose

Break the task into **independent, non-overlapping workstreams**. Good workstreams:

- Touch different files (minimize file conflicts)
- Can be verified independently before integration
- Have clear, narrow scope — one concern per stream
- Sum to the complete task (no gaps)

**Decomposition rules:**
1. **Max 4 workstreams** — more than 4 causes more overhead than benefit.
2. **Each stream must be independently valid** — no broken imports, no partial references.
3. **Order dependencies explicitly** — if Stream B needs Stream A's output, mark it.
4. **Name each stream** — short, descriptive (e.g., "API endpoint", "React component", "Unit tests").

Output a **Workstream Plan** before proceeding:

```
Workstream Plan
───────────────
Stream A: [name] — [what it does, which files it touches]
Stream B: [name] — [what it does, which files it touches]
Stream C: [name] — [what it does, which files it touches]

Dependencies: B requires A's output.
Parallel: A and C can run simultaneously.
```

**If any streams conflict** (both touch the same critical shared file): merge them or sequence them. Flag the dependency explicitly.

---

## Step 2: Confirm Plan (AskUserQuestion)

Before dispatching, ask:

> "I'll run [N] workstreams in parallel:
> - Stream A: [name] — [scope]
> - Stream B: [name] — [scope]
> ...
> Ready to proceed? Or adjust the plan?"

Wait for confirmation. If the user adjusts, revise and re-confirm. If the user says "go" or "yes", dispatch immediately.

**Only skip confirmation when the user has explicitly described exactly what to parallelize** (e.g., "split this into frontend component and its tests") **and the task involves no destructive changes** (no deletions, no database migrations, no files being moved). Always confirm for refactors touching more than 5 files, any migration, or any deletion.

---

## Step 3: Dispatch Workstreams

### Parallel execution

Use Task for independent streams, sequential Bash for dependent streams.

For each workstream, write a focused sub-task prompt that includes:
- **Exact scope**: which files to create/edit, what to implement
- **Constraints**: do not touch outside this scope
- **Verification**: what "done" looks like for this stream
- **Context**: repo language, framework, key patterns from CLAUDE.md

**Task prompt template:**
```
You are working on [Stream Name] for the /cowork orchestration.

Scope: [exact files and changes]
Constraints: Only modify the files listed above. Do not refactor adjacent code.
Goal: [specific outcome]
Done when: [concrete verification — e.g., "tests pass", "endpoint returns 200", "component renders"]

Context: [language, framework, key conventions from CLAUDE.md]
```

### Progress board

After dispatching, maintain a live progress board:

```
Workstream Progress
───────────────────
✓ Stream A: API endpoint          [DONE — POST /api/items returns 201]
⟳ Stream B: React component       [IN PROGRESS]
⟳ Stream C: Unit tests            [IN PROGRESS — blocked on Stream A]
○ Stream D: Documentation         [PENDING]
```

Update the board after each stream completes.

---

## Step 4: Mode-Specific Workflows

### Build mode (`/cowork build`)

Typical split for a full-stack feature:

| Stream | Scope | Verification |
|--------|-------|--------------|
| **Backend** | Models, migrations, services, API endpoints | `bin/rails test` or equivalent |
| **Frontend** | React/Vue/Svelte components, CSS | Renders in browser, no console errors |
| **Tests** | Integration + unit tests for both layers | Test suite passes |
| **Docs** | README update, inline comments, API docs | Diff shows docs match implementation |

If the feature is smaller (< 5 files), collapse to 2 streams: implementation + tests.

After all streams complete: run the full test suite once to confirm integration:

```bash
# Run tests (adapt to the project's test command from CLAUDE.md)
git diff --stat   # sanity-check the combined diff
```

If tests fail: identify which stream caused the failure, apply a fix, re-run.

---

### Debug mode (`/cowork debug`)

Generate 3–4 competing hypotheses for the bug. Each hypothesis becomes a stream:

**Hypothesis generation:**
1. Read the error message/stack trace carefully
2. Identify the 3 most likely root causes (different layers/components)
3. Name each stream by its hypothesis: "DB query timeout", "Cache staleness", "Race condition"

**Per-stream investigation:**
- Check the relevant code path
- Look for the specific failure pattern
- Write repro steps if found
- Output: `CONFIRMED: [evidence]` or `RULED OUT: [why]`

**Convergence:** After all streams report:
- If one is CONFIRMED: proceed with the fix
- If multiple are CONFIRMED: they may be related — investigate the intersection
- If all are RULED OUT: generate a second round of hypotheses

**Fix:** Once root cause is confirmed, implement the fix in a single focused edit. Then run the relevant test suite to verify.

---

### Review mode (`/cowork review`)

Multi-lens review runs four specialized sub-reviewers in parallel:

| Stream | Focus | What to look for |
|--------|-------|-----------------|
| **Security** | Trust boundaries, injection, auth, data exposure | SQL injection, XSS, SSRF, over-exposed fields, unvalidated inputs |
| **Performance** | N+1 queries, missing indexes, sync in hot paths | `each` inside loops, missing `.includes`, no pagination, blocking I/O |
| **Correctness** | Logic bugs, race conditions, edge cases | Off-by-one, nil handling, concurrent writes, missing rollback |
| **Observability** | Logging, error handling, metrics | Silent failures, bare `rescue`, uncaught exceptions, no audit trail |

First, get the diff:

```bash
git fetch origin main --quiet
git diff origin/main
```

Dispatch all four reviewers on the same diff simultaneously.

**Output format** — aggregate results into a single report:

```
/cowork review — [N] issues found ([X] critical, [Y] informational)

CRITICAL
────────
[file:line] [Security] SQL injection via unsanitized user.name in search query
  Fix: Use parameterized query: User.where("name = ?", params[:name])

INFORMATIONAL
─────────────
[file:line] [Performance] N+1 query in OrdersController#index — add .includes(:items)
[file:line] [Observability] Rescue block swallows error silently — add logger.error(e)
```

**If CRITICAL issues found:** for each one, use AskUserQuestion with: the problem, your recommended fix, and options (A: Fix now, B: Acknowledge, C: False positive).

This mode integrates with and extends `/review`. If the user wants the full gstack review checklist applied, resolve the skill dir (see Important Rules) and run `cat "$_SKILL_DIR/review/checklist.md"` and apply it on top.

---

### Test mode (`/cowork test`)

Split test generation across modules, services, or layers:

1. **Identify untested code**: `git diff main --name-only` → find new/changed files
2. **Group by module**: e.g., `app/models/`, `app/services/`, `app/controllers/`
3. **One stream per module group**: each stream writes tests for its group
4. **Constraints per stream**: follow existing test patterns in that directory

After all streams complete:

```bash
# Run the full test suite
git diff --stat test/   # review what was written
```

If any tests fail (because they assert wrong behavior): fix the test, not the code.

---

### Refactor mode (`/cowork refactor`)

For large refactors (renaming, extracting concerns, migrating patterns):

1. **Identify all call sites**: `grep -r "OldName" --include="*.rb" -l` (adapt to language)
2. **Group by file cluster**: related files in the same module/directory
3. **One stream per cluster**: each stream handles its cluster's rename/refactor
4. **Final stream**: update imports, re-export, integration glue

**Critical rule:** Each stream must leave the codebase in a valid state — no dangling imports, no undefined symbols.

After all streams: run static analysis + tests:

```bash
git diff --stat        # verify expected file count changed
# Run type checker / linter if available
```

---

### Plan mode (`/cowork plan`)

Run both gstack planning modes in parallel, then synthesize:

**Stream A: CEO Review** — resolve the skill dir (see Important Rules), then read `"$_SKILL_DIR/plan-ceo-review/SKILL.md"` and apply SCOPE EXPANSION mode to the task
**Stream B: Eng Review** — resolve the skill dir (same method), then read `"$_SKILL_DIR/plan-eng-review/SKILL.md"` and apply the full engineering deep-dive

After both complete, synthesize into a unified plan:
- CEO review sets the product direction and ambition level
- Eng review sets the technical architecture and implementation plan
- Resolve any conflicts between the two (e.g., CEO wants X, eng review reveals X is risky — surface the trade-off)

Output: one unified plan document with both the product vision (from CEO mode) and the technical specification (from eng mode).

---

## Step 5: Integration

After all streams complete:

1. **Review the combined diff:**
   ```bash
   git diff --stat
   git diff
   ```

2. **Check for conflicts:** Are there any files touched by multiple streams? If yes, read each conflicting file and verify consistency.

3. **Run the test suite:**
   Adapt to the project. Look for test commands in CLAUDE.md, Makefile, or package.json.
   ```bash
   # Find the test command
   cat CLAUDE.md 2>/dev/null | grep -i "test\|spec\|check" | head -10
   ```

4. **If tests fail:** Identify the failing stream, use Edit to apply a targeted fix to the specific file, then re-run only that test file.

5. **Integration smoke test:** For build/refactor modes, if a browse binary is available, run a quick smoke check:
   ```bash
   _ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
   B=""
   [ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/browse/dist/browse" ] && B="$_ROOT/.claude/skills/gstack/browse/dist/browse"
   [ -z "$B" ] && B=~/.claude/skills/gstack/browse/dist/browse
   if [ -x "$B" ]; then
     $B goto http://localhost:3000
     $B snapshot -i
     $B console
   fi
   ```

---

## Step 6: Output

Deliver a final summary:

```
/cowork complete — [mode] in [N] workstreams

Streams executed:
✓ Stream A: [name] — [outcome in one line]
✓ Stream B: [name] — [outcome in one line]
✓ Stream C: [name] — [outcome in one line]

Integration:
- Tests: [pass/fail counts]
- Diff: [N files changed, +X/-Y lines]
- Issues resolved: [any integration conflicts fixed]

Next steps:
- /review — run paranoid code review on the full diff
- /qa — smoke test the feature in a browser
- /ship — merge main, run tests, open PR
```

---

## Important Rules

- **Never overlap workstream scopes.** If two streams must touch the same file, sequence them — stream B starts only after stream A has fully completed its work on that file (use blocking Task execution, not parallel dispatch).
- **Each stream must be independently valid.** No broken state mid-execution.
- **Always confirm the plan** before dispatching (unless simple 2-stream, non-destructive task).
- **Test after every mode.** Integration bugs are the most expensive kind.
- **Use gstack skills by reading their SKILL.md files.** Resolve paths using: `_SKILL_DIR=$( [ -d "$_ROOT/.claude/skills/gstack" ] && echo "$_ROOT/.claude/skills/gstack" || echo "$HOME/.claude/skills/gstack" )`. Then: `cat "$_SKILL_DIR/review/checklist.md"` or `cat "$_SKILL_DIR/plan-ceo-review/SKILL.md"`.
- **The goal is speed without chaos.** Parallelism is a tool, not a goal. If the task is inherently sequential, say so and execute it sequentially.

---

## Tips

1. **Start narrow, go wide.** 2 streams for a small feature, 4 for a large one. Never more than 4.
2. **Name streams by concern, not by file.** "Auth logic" is a good stream; "user.rb" is not.
3. **Put the riskiest work in Stream A.** Fail fast — if the core logic is broken, you want to know before running tests and docs.
4. **Use `/cowork review` before `/ship`.** Multi-lens review catches more than single-pass review.
5. **`/cowork plan` → implement → `/cowork review` → `/ship` is the full cycle.** Start in plan mode for any non-trivial feature.
6. **Debug mode is underused.** When stuck on a bug for > 15 minutes, run `/cowork debug <error>` to investigate 3 hypotheses at once.
