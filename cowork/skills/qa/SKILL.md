---
name: qa
description: Systematic QA testing of web applications. Use when asked to "qa", "QA", "test this site", "find bugs", "dogfood", "smoke test", or "run QA". Four modes — diff-aware (automatic on feature branches), full (systematic exploration), quick (30-second smoke test), regression (compare against baseline). Produces a structured report with health score, screenshots, and repro steps. Requires gstack browser binary.
argument-hint: "[URL] [--quick] [--regression <baseline.json>]"
---

# /qa

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../../CONNECTORS.md).

You are a QA engineer. Test web applications like a real user — click everything, fill every form, check every state. Produce a structured report with evidence.

## Usage

```
/qa                        ← auto-detects feature branch, tests changed pages
/qa https://myapp.com      ← full systematic QA
/qa --quick                ← 30-second smoke test
/qa --regression baseline  ← compare against previous run
```

## Setup

The `/qa` skill requires the gstack browser binary. See the `/browse` skill for setup instructions.

**Before any browse command, verify the binary:**

```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/browse/dist/browse" ] && B="$_ROOT/.claude/skills/gstack/browse/dist/browse"
[ -z "$B" ] && B=~/.claude/skills/gstack/browse/dist/browse
if [ -x "$B" ]; then echo "READY: $B"; else echo "NEEDS_SETUP"; fi
```

If `NEEDS_SETUP`: Run `cd ~/.claude/skills/gstack && ./setup`

## Parameters

| Parameter | Default | Override |
|-----------|---------|----------|
| Target URL | Auto-detect or required | `https://myapp.com`, `http://localhost:3000` |
| Mode | diff-aware (on feature branch) or full | `--quick`, `--regression <path>` |
| Output dir | `.gstack/qa-reports/` | Specify in request |
| Scope | Full app or diff-scoped | "Focus on the billing page" |
| Auth | None | "Sign in to user@example.com" |

## Modes

### Diff-aware (automatic when on a feature branch with no URL)

This is the **primary mode** for developers verifying their work.

1. **Analyze the branch diff** to understand what changed:
   ```bash
   git diff main...HEAD --name-only
   git log main..HEAD --oneline
   ```
   If **GitHub is connected**: pull the PR diff automatically.

2. **Identify affected pages/routes** from changed files:
   - Controller/route files → URL paths they serve
   - View/template/component files → pages that render them
   - Model/service files → pages that use those models
   - CSS files → pages that include those stylesheets
   - API endpoints → test directly with JS fetch

3. **Detect the running app** — check common local dev ports:
   ```bash
   $B goto http://localhost:3000 2>/dev/null && echo "Found :3000" || \
   $B goto http://localhost:4000 2>/dev/null && echo "Found :4000" || \
   $B goto http://localhost:8080 2>/dev/null && echo "Found :8080"
   ```
   If nothing works, ask the user for the URL.

4. **Test each affected page** — navigate, screenshot, check console, test interactions.

5. **Report findings** scoped to the branch changes with screenshot evidence.

### Full (default when URL is provided)

Systematic exploration. Visit every reachable page. Document 5-10 well-evidenced issues. Produce health score. Takes 5-15 minutes depending on app size.

### Quick (`--quick`)

30-second smoke test. Visit homepage + top 5 navigation targets. Check: page loads? Console errors? Broken links? Produce health score.

### Regression (`--regression <baseline>`)

Run full mode, then load `baseline.json` from a previous run. Diff which issues are fixed, which are new, and what the score delta is.

---

## Workflow

### Phase 1: Initialize

1. Find browse binary (see Setup)
2. Create output directories: `mkdir -p .gstack/qa-reports/screenshots`
3. Note start time for duration tracking

### Phase 2: Authenticate (if needed)

**If auth credentials specified:**
```bash
$B goto <login-url>
$B snapshot -i
$B fill @e3 "user@example.com"
$B fill @e4 "[REDACTED]"
$B click @e5
$B snapshot -D                    # verify login succeeded
```

**If cookie file provided:**
```bash
$B cookie-import cookies.json
$B goto <target-url>
```

If 2FA/OTP required: Ask for the code and wait.

### Phase 3: Orient

Map the application structure:
```bash
$B goto <target-url>
$B snapshot -i -a -o ".gstack/qa-reports/screenshots/initial.png"
$B links                          # map navigation
$B console --errors               # any errors on landing?
```

**Detect framework:**
- `__next` in HTML → Next.js
- `csrf-token` meta tag → Rails
- `wp-content` in URLs → WordPress
- Client-side routing → SPA (use `snapshot -i` for nav elements)

### Phase 4: Explore

At each page:
```bash
$B goto <page-url>
$B snapshot -i -a -o ".gstack/qa-reports/screenshots/page-name.png"
$B console --errors
```

Per-page exploration checklist:
1. **Visual scan** — layout issues, overflow, missing content
2. **Interactive elements** — click buttons, links, controls
3. **Forms** — fill and submit, test empty/invalid/edge cases
4. **Navigation** — all paths in and out
5. **States** — empty state, loading, error, overflow
6. **Console** — new JS errors after interactions?

### Phase 5: Document Issues

For each issue found, capture:
- **Screenshot** (annotated if interactive)
- **Console errors** (if applicable)
- **Repro steps** (numbered, specific)
- **Severity** (critical/high/medium/low)

Issue taxonomy:
- **Visual** — layout breaks, overflow, missing images, inconsistent styling
- **Functional** — broken interactions, wrong behavior, crashes
- **UX** — confusing flows, misleading copy, missing feedback
- **Content** — wrong text, broken links, outdated info
- **Performance** — slow loads, large assets, console warnings
- **Console** — JS errors, network failures, CSP violations
- **Accessibility** — missing labels, low contrast, keyboard traps

### Phase 6: Wrap Up

Calculate health score:
- Visual integrity: 0-25 pts
- Functional correctness: 0-30 pts
- Console cleanliness: 0-15 pts
- Navigation: 0-10 pts
- Performance: 0-10 pts
- Content quality: 0-10 pts

Final report format:
```markdown
# QA Report — [App Name]
**Date:** [date] | **Mode:** [mode] | **Duration:** [time] | **Health Score:** [N]/100

## Summary
[2-3 sentence executive summary]

## Issues Found (N total)

### Critical (N)
#### [Issue Title]
**Severity:** Critical | **Page:** [URL]
[Description]
**Steps to reproduce:**
1. [Step]
2. [Step]
**Expected:** [behavior]
**Actual:** [behavior]
**Evidence:** [screenshot path]

### High / Medium / Low
[same format]

## Health Score Breakdown
| Category | Score | Notes |
|----------|-------|-------|
| Visual | N/25 | ... |
```

## If Connectors Available

If **GitHub** is connected:
- Pull the PR diff for diff-aware mode automatically
- Post QA report as a PR comment
- Link issues to the PR

## Tips

1. **Screenshot first, investigate second.** Capture the state before interacting.
2. **Depth over breadth.** 5 well-documented issues > 15 vague ones.
3. **Every issue needs a repro.** If you can't reproduce it consistently, it's not documented.
4. **Use `snapshot -D` to verify changes.** Baseline → action → diff shows exactly what changed.
