# gstack Cowork — Step-by-Step User Guide

Everything you need to go from zero to a fully-connected engineering workflow assistant in under 15 minutes.

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [Install the Plugin](#2-install-the-plugin)
3. [First-Use Checklist](#3-first-use-checklist)
4. [Skill Walkthroughs](#4-skill-walkthroughs)
   - [/plan-ceo-review](#41-plan-ceo-review)
   - [/plan-eng-review](#42-plan-eng-review)
   - [/review](#43-review)
   - [/ship](#44-ship)
   - [/retro](#45-retro)
   - [/browse](#46-browse)
   - [/qa](#47-qa)
   - [/setup-browser-cookies](#48-setup-browser-cookies)
5. [Connect Your Tools (Supercharged Mode)](#5-connect-your-tools-supercharged-mode)
6. [Common Workflow Recipes](#6-common-workflow-recipes)
7. [Troubleshooting](#7-troubleshooting)

---

## 1. Prerequisites

| Requirement | Why | How to get it |
|---|---|---|
| Claude Cowork account | Runs the plugin | [claude.com/product/cowork](https://claude.com/product/cowork) |
| **Bun** v1.0+ *(browser skills only)* | Builds the headless browser binary | `curl -fsSL https://bun.sh/install \| bash` |
| macOS or Linux, x64 or arm64 *(browser skills only)* | Binary compilation target | — |

> **No binary needed for** `/plan-ceo-review`, `/plan-eng-review`, `/review`, `/ship`, and `/retro`. Those five skills work the moment the plugin is installed.

---

## 2. Install the Plugin

### Option A — One-liner (recommended)

Open Claude Cowork and run:

```
/plugins add garrytan/gstack/cowork
```

### Option B — Manual install

Paste this prompt into Cowork:

> Install the gstack plugin: download `https://github.com/garrytan/gstack` and add the `cowork/` directory as a local plugin.

### Verify the install

After installing, type `/` in any Cowork conversation. You should see these commands in the autocomplete list:

```
/plan-ceo-review
/plan-eng-review
/review
/ship
/browse
/qa
/setup-browser-cookies
/retro
```

---

## 3. First-Use Checklist

Work through this list top-to-bottom. Skip the browser section if you only need the conversational skills.

### ✅ Conversational skills (no setup needed)

- [ ] Plugin installed (Step 2 above)
- [ ] Run `/plan-ceo-review` — type a one-sentence feature idea to smoke-test it
- [ ] Run `/review` — paste a small diff to verify the skill loads

### ✅ Browser skills (one-time build, ~10 seconds)

- [ ] Bun installed (`bun --version` should print a version number)
- [ ] Clone and build the binary:

  ```bash
  git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
  cd ~/.claude/skills/gstack && ./setup
  ```

- [ ] Verify the binary is ready:

  ```bash
  ~/.claude/skills/gstack/browse/dist/browse --version
  ```

  Expected output: a version string like `browse/1.x.x`.

- [ ] Run `/browse goto https://example.com` — you should get a page snapshot back

### ✅ Connectors (optional, for supercharged mode)

- [ ] GitHub connected → `/review` and `/ship` can pull PR diffs automatically
- [ ] Linear connected → `/review` can link findings to tickets
- [ ] Notion connected → `/plan-eng-review` can read your ADRs
- [ ] Slack connected → `/retro` can post summaries to your team channel

See [Section 5](#5-connect-your-tools-supercharged-mode) for setup steps.

---

## 4. Skill Walkthroughs

### 4.1 `/plan-ceo-review`

**What it does:** Founder/CEO-mode plan review. Challenges premises, searches for the 10-star product version, catches every landmine before a line of code is written. Choose from three modes:
- **Scope Expansion** — push the idea to its ambitious limit
- **Hold Scope** — make the existing plan bulletproof
- **Scope Reduction** — strip to the minimum viable version

**Step-by-step:**

1. Open a new Cowork conversation.
2. Type `/plan-ceo-review` and press Enter.
3. When prompted, describe your plan. You can paste a written doc, a few bullet points, or a PR/issue link (if GitHub is connected).
4. Claude will run a **System Audit** (pulls recent commits and open PRs if GitHub is connected), then open **Step 0: Scope Challenge**.
5. Claude presents three mode options. **Pick one** — e.g., reply `2` or `Hold Scope`.
6. Work through each review section interactively. Claude will stop and ask one question at a time; answer each before it continues.
7. At the end, Claude delivers:
   - A one-paragraph summary
   - Open questions to resolve before coding
   - A recommended next step

**Example input:**
```
/plan-ceo-review

We want to add a "team inbox" feature to our SaaS product where multiple teammates can see and triage incoming support tickets together.
```

---

### 4.2 `/plan-eng-review`

**What it does:** Engineering-lead-mode plan review. Validates architecture, data flow, edge cases, test coverage, and performance before any code is written.

**Step-by-step:**

1. Type `/plan-eng-review`.
2. Paste or describe your implementation plan. Include: what you're building, the files you'll touch, any new services or DB tables, and your testing approach.
3. Claude runs **Step 0: Scope Challenge** and asks you to pick a review depth:
   - `Scope Reduction` — plan is overbuilt, propose a leaner version
   - `Big Change` — full interactive walkthrough section-by-section
   - `Small Change` — compressed single-pass review
4. Work through each section (Architecture → Code Quality → Tests → Performance), answering Claude's questions one at a time.
5. Receive a **Completion Summary** with open questions and a recommended first commit.

**Tips:**
- Share constraints upfront: "We need to ship in 2 days" or "We can't change the schema."
- Paste relevant existing code so Claude reviews in context, not in isolation.

---

### 4.3 `/review`

**What it does:** Paranoid code review from a staff engineer perspective. Two-pass review: critical issues (SQL safety, race conditions, LLM trust boundaries, auth) first, then informational issues.

**Step-by-step:**

1. Type `/review` followed by one of:
   - A PR URL: `/review https://github.com/org/repo/pull/123`
   - A pasted diff (paste directly after the command)
   - A file path: `/review src/auth/login.ts`

2. *(If GitHub is connected)* Claude pulls the diff automatically.

3. Claude runs a **two-pass review**:
   - **Pass 1 — Critical:** Security, data integrity, race conditions, and trust boundary violations. These block merge.
   - **Pass 2 — Informational:** Code quality, naming, DRY violations, test gaps. These are suggestions.

4. *(If Greptile is connected)* Claude also runs a Greptile analysis (Step 4) and filters out false positives from the codebase history before surfacing findings.

5. Each finding includes: location, severity, explanation, and a concrete fix.

**Example:**
```
/review https://github.com/myorg/myapp/pull/42
```

**Standalone (no GitHub):**
```diff
/review

--- a/src/users.js
+++ b/src/users.js
@@ -10,7 +10,7 @@ function getUser(id) {
-  const query = `SELECT * FROM users WHERE id = ${id}`;
+  const query = `SELECT * FROM users WHERE id = ?`;
```

---

### 4.4 `/ship`

**What it does:** Fully automated release engineer — syncs main, runs tests, bumps version, updates changelog, commits, pushes, and opens a PR. Use this when your branch is ready to land, not when you're still deciding what to build.

**Step-by-step:**

1. Make sure your feature branch is checked out and all changes are committed locally.
2. Type `/ship`.
3. Claude runs the full ship workflow non-interactively:
   1. Fetches and merges latest `main`
   2. Runs the test suite
   3. Resolves any outstanding review comments (if GitHub is connected)
   4. Bumps the version and updates `CHANGELOG.md`
   5. Commits, pushes, and opens a PR

4. If any step fails (merge conflict, test failure, etc.), Claude stops, reports the exact error, and asks how to proceed.

> **Note:** `/ship` is opinionated and non-interactive by design. If you want to review each step before it runs, use `/plan-eng-review` instead.

**Prerequisites for full automation:**
- GitHub connected (for automatic PR creation and comment resolution)
- A test command configured in `package.json` or equivalent

---

### 4.5 `/retro`

**What it does:** Engineering retrospective — analyzes commit history, PR patterns, and code metrics. Team-aware: per-person praise and growth areas for every contributor.

**Step-by-step:**

1. Type `/retro` with an optional time window:
   ```
   /retro              ← last 7 days (default)
   /retro 24h          ← last 24 hours
   /retro 14d          ← last 14 days
   /retro 30d          ← last 30 days
   /retro compare      ← current period vs. the same-length prior period
   /retro compare 14d  ← compare 14-day windows explicitly
   ```

2. *(If GitHub is connected)* Claude pulls commit history, PR merges, and review activity automatically.

3. *(Standalone)* Claude will ask you to describe the week's work or paste commit logs.

4. Claude produces a structured report with:
   - Velocity metrics (commits, LOC, PRs merged)
   - Code quality signals (test coverage changes, review turnaround)
   - Per-contributor section: what they crushed, what to grow next
   - Team-level themes and recommended focus for next sprint

5. *(If Slack is connected)* Claude offers to post the summary to a team channel.

---

### 4.6 `/browse`

**What it does:** Gives Claude a persistent headless Chromium browser. Navigate URLs, click elements, fill forms, take screenshots, verify page state. Requires the gstack binary (see [Section 3](#3-first-use-checklist)).

**Step-by-step:**

1. Verify setup:
   ```bash
   ~/.claude/skills/gstack/browse/dist/browse --version
   ```

2. Use natural language or explicit commands:

   ```
   /browse goto https://myapp.com/login
   /browse snapshot -i
   /browse screenshot /tmp/before.png
   ```

3. Claude auto-starts the browser on the first call (~3 seconds), then each subsequent command takes ~100–200 ms. The browser state (cookies, tabs, sessions) persists for 30 minutes of idle time.

**Common commands:**

| Goal | Command |
|---|---|
| Navigate to a URL | `/browse goto https://example.com` |
| Snapshot the current page | `/browse snapshot -i` |
| Take a screenshot | `/browse screenshot /tmp/shot.png` |
| Click a button | `/browse click "Sign in"` |
| Fill a form field | `/browse fill "#email" user@example.com` |
| Check page title | `/browse eval "document.title"` |

---

### 4.7 `/qa`

**What it does:** Systematic QA testing of web apps. Four modes: diff-aware (automatic on feature branches), full exploration, quick 30-second smoke test, and regression comparison. Requires the gstack binary.

**Step-by-step:**

1. Choose your mode:

   | Situation | Command |
   |---|---|
   | On a feature branch, test what changed | `/qa` |
   | Full app QA | `/qa https://myapp.com` |
   | Quick sanity check | `/qa --quick` |
   | Compare against a previous run | `/qa --regression .gstack/qa-reports/baseline.json` |

2. *(Diff-aware mode)* Claude analyzes your git diff to identify changed pages, then tests only those.

3. *(If GitHub is connected)* Claude pulls the PR diff to determine the scope automatically.

4. Claude navigates the app, clicks through flows, fills forms, checks states, and takes screenshots of any issues found.

5. Receives a structured **QA Report**:
   - Health score (0–100)
   - Issues list with severity, repro steps, and screenshots
   - Recommendations

**Example:**
```
/qa https://staging.myapp.com

Focus on the checkout flow — we changed the pricing logic in this branch.
```

---

### 4.8 `/setup-browser-cookies`

**What it does:** Imports cookies from your real browser (Chrome, Arc, Brave, Edge, Comet) into the headless Chromium session. Lets you QA authenticated pages without logging in manually each time.

**Step-by-step:**

1. Make sure the gstack binary is installed (see [Section 3](#3-first-use-checklist)).

2. Run the interactive picker (recommended for first-time use):
   ```
   /setup-browser-cookies
   ```
   This opens a browser UI where you pick which browser and domain to import from.

3. Or import directly for a known domain:
   ```
   /setup-browser-cookies chrome --domain .github.com
   /setup-browser-cookies arc --domain staging.myapp.com
   /setup-browser-cookies brave --domain myapp.com
   ```

4. After import, `/browse` and `/qa` will automatically use the imported cookies — no login required in the headless session.

> **Tip:** Re-run this command any time your real-browser session expires or you switch accounts.

---

## 5. Connect Your Tools (Supercharged Mode)

Every skill works standalone. These connections make them dramatically more powerful.

### How to Connect

1. In Cowork, go to **Customize → Plugins → gstack → Connectors**
2. Toggle on each integration you want
3. Authorize via OAuth when prompted

### What Each Connector Enables

| Connector | Enables |
|---|---|
| **GitHub** | `/review` pulls PR diffs automatically • `/ship` creates PRs and resolves comments • `/retro` reads full commit history and PR activity • `/qa` reads the PR diff for diff-aware testing • `/review` and `/ship` triage Greptile bot comments |
| **GitLab** | Same as GitHub for GitLab-hosted repositories |
| **Greptile** | Direct code intelligence MCP — semantic code search and cross-file analysis for `/review` and `/ship` |
| **Linear** | `/review` can link findings directly to Linear tickets • `/ship` can close related issues on merge |
| **Notion** | `/plan-eng-review` reads prior ADRs, architecture docs, and runbooks • `/review` applies your team's documented coding standards |
| **Slack** | `/retro` posts summaries to your team channel • `/qa` shares QA reports |

### Greptile — Step-by-Step Setup

Greptile gives `/review` and `/ship` a superpower: every PR is automatically reviewed by an AI bot (`greptile-apps[bot]`), and gstack triages those comments — classifying each as `VALID`, `ALREADY-FIXED`, or `FALSE-POSITIVE` and suppressing known noise.

There are two parts to the setup:

#### Part 1 — Install the Greptile GitHub App (required for bot triage)

This is what makes Greptile post review comments on your PRs. It does **not** require a Greptile API key — just your GitHub connection.

1. Go to **[app.greptile.com](https://app.greptile.com)** and sign in with GitHub.
2. Click **Add Repository** and select every repo you want reviewed.
3. Wait ~2–5 minutes for Greptile to index the codebase. You'll get an email when it's ready.
4. From now on, every new PR in those repos will automatically get Greptile review comments.

> **Verify:** Open any PR in the repo and look for a comment from `greptile-apps[bot]`. If you see one, the GitHub App is working.

#### Part 2 — Connect the Greptile MCP (optional, for direct code intelligence)

This unlocks semantic code search and cross-file analysis directly inside Cowork sessions.

1. Go to **[app.greptile.com/settings/api](https://app.greptile.com/settings/api)** and copy your API key.
2. In Cowork, open **Customize → Plugins → gstack → Connectors**.
3. Find **Greptile** in the connectors list and toggle it on.
4. Paste your Greptile API key when prompted.

Cowork will connect to `https://api.greptile.com/mcp` using that key. Once connected, `/review` and `/ship` can query the Greptile codebase index directly — not just read bot comments.

#### Part 3 — Verify the full integration

1. Make sure your GitHub connection is active (Step 2 of this guide).
2. Open a PR in a Greptile-enrolled repo and wait for the bot comment to appear.
3. Run `/review <PR URL>` in Cowork.
4. After the two-pass review, you should see:
   ```
   ## Greptile Review: N comments (X valid, Y fixed, Z false positive)
   - [VALID] src/auth.ts:42 — Missing null check on user.email — <link>
   - [FALSE POSITIVE] src/utils.ts:17 — Flagged unused import (already removed in this PR) — <link>
   ```
5. For each VALID comment, Claude asks: **A) Fix it now, B) Acknowledge, C) False positive — skip**.

#### Part 4 — False positive suppression

When you mark a comment as a false positive (option C), gstack saves it to `~/.gstack/greptile-history.md`:

```
2026-03-14 | myorg/myapp | fp | src/utils/*.ts | style
```

Future `/review` and `/ship` runs automatically skip any comment matching that entry. You only see real issues.

To clear the suppression history: `rm ~/.gstack/greptile-history.md`

---

## 6. Common Workflow Recipes

### Recipe 1: Review and ship a feature branch end-to-end

```
# 1. Review the code before merging
/review https://github.com/myorg/myapp/pull/99

# 2. QA the staging deploy
/qa https://staging.myapp.com

# 3. Ship once review is green
/ship
```

### Recipe 2: Plan a new feature before writing any code

```
# CEO mode — is this the right thing to build?
/plan-ceo-review
I want to add SSO login to our B2B SaaS product.

# Eng mode — how should we build it?
/plan-eng-review
We'll use the WorkOS SDK. Here's the schema change + flow...
```

### Recipe 3: Authenticated QA testing

```
# One-time: import your browser cookies
/setup-browser-cookies chrome --domain .myapp.com

# Now QA authenticated pages
/qa https://myapp.com/dashboard
/browse goto https://myapp.com/settings/billing
```

### Recipe 4: Weekly team retro

```
# Run after every sprint
/retro 14d

# Compare to prior sprint
/retro compare 14d
```

### Recipe 5: Quick smoke test before a deploy

```
/qa --quick https://staging.myapp.com
```

---

## 7. Troubleshooting

### Plugin commands don't appear after install

- Refresh your Cowork session.
- Go to **Customize → Plugins** and confirm `gstack` is listed and enabled.
- Re-run the install command from [Section 2](#2-install-the-plugin).

### `/browse`, `/qa`, or `/setup-browser-cookies` reports `NEEDS_SETUP`

The gstack binary hasn't been built. Run:

```bash
git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup
```

If the clone already exists, run the setup again from inside the directory:

```bash
cd ~/.claude/skills/gstack && ./setup
```

### Bun not found when running `./setup`

Install Bun first:

```bash
curl -fsSL https://bun.sh/install | bash
# Reload your shell, then re-run ./setup
```

### `/ship` fails with a merge conflict

Claude will stop and report the conflicting files. Resolve the conflicts manually, commit the resolution, then re-run `/ship`.

### `/review` shows "no diff found" in standalone mode

Either paste the diff directly after the command, provide a file path, or connect GitHub so Claude can pull the PR automatically.

### `/retro` shows no commits

- If using GitHub connector: confirm the repo is authorized.
- Standalone: paste your `git log` output or describe the week's work when Claude prompts you.

### `/setup-browser-cookies` fails to read browser cookies

- macOS: Claude needs **Full Disk Access** in System Settings → Privacy & Security.
- Chrome/Brave: close the browser before importing (the SQLite cookie DB is locked while Chrome is open).

### A connector shows as "connected" but skills still run in standalone mode

Each skill falls back to standalone automatically if the connector returns an error. Check:
1. The OAuth token hasn't expired — re-authorize in **Customize → Plugins → gstack → Connectors**.
2. The connector has the right scopes (e.g., GitHub needs `repo` access for private repos).

---

*For the full technical reference, see [DESIGN.md](DESIGN.md). For the list of all supported connectors, see [CONNECTORS.md](CONNECTORS.md).*
