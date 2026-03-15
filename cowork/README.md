# gstack — Cowork Plugin

An engineering workflow plugin for [Claude Cowork](https://claude.com/product/cowork) that turns Claude into a team of specialists you can summon on demand.

Eight opinionated skills covering plan review, code review, one-command shipping, headless browser QA testing, and team retrospectives — all as slash commands.

## Installation

```bash
claude plugins add garrytan/gstack/cowork
```

Or copy-paste this prompt into Cowork to install manually:

> Install the gstack plugin: download `https://github.com/garrytan/gstack` and add the `cowork/` directory as a local plugin.

**New to gstack?** See the [Step-by-Step User Guide](GUIDE.md) for a complete walkthrough — from installation through connecting your tools and running every skill.

## Commands

Explicit workflows you invoke with a slash command:

| Command | Mode | What it does |
|---------|------|--------------|
| `/plan-ceo-review` | Founder / CEO | Rethink the problem. Find the 10-star product hiding inside the request. Three modes: Scope Expansion, Hold Scope, Scope Reduction. |
| `/plan-eng-review` | Eng manager / tech lead | Lock in architecture, data flow, diagrams, edge cases, and tests. Interactive walkthrough with opinionated recommendations. |
| `/review` | Paranoid staff engineer | Find the bugs that pass CI but blow up in production. Works with a PR URL, pasted diff, or file path. |
| `/ship` | Release engineer | Sync main, run tests, resolve review comments, push, open PR. For a ready branch, not for deciding what to build. |
| `/browse` | QA engineer | Give Claude eyes. Navigate URLs, interact with elements, take screenshots, verify page state. Requires gstack binary (see Setup). |
| `/qa` | QA lead | Systematic QA testing. Analyzes your diff, identifies affected pages, and tests them. Also: full exploration, quick smoke test, regression mode. |
| `/setup-browser-cookies` | Session manager | Import cookies from your real browser (Chrome, Arc, Brave, Edge) into the headless session. |
| `/retro` | Engineering manager | Team-aware retrospective: deep-dive + per-person praise and growth opportunities for every contributor. |

All commands work **standalone** (describe your plan/code, paste a diff) and get **supercharged** with MCP connectors.

## Skills

Domain knowledge Claude uses automatically when relevant:

| Skill | Description |
|-------|-------------|
| `plan-ceo-review` | CEO/founder perspective on plans — challenge premises, find the 10-star product |
| `plan-eng-review` | Engineering lead perspective — architecture, diagrams, edge cases, test coverage |
| `review` | Paranoid code review — SQL safety, race conditions, LLM trust boundaries, security |
| `ship` | Ship workflow — sync, test, bump version, changelog, push, PR |
| `browse` | Headless browser automation — navigate, click, fill, screenshot, assert |
| `qa` | Systematic QA testing — explore, document issues, health score |
| `setup-browser-cookies` | Import real browser cookies for authenticated testing |
| `retro` | Engineering retrospectives — metrics, patterns, per-person insights |

## Example Workflows

### Plan Review (CEO Mode)

```
/plan-ceo-review
```

Describe your feature idea. Get pushed to find the 10-star product, not just implement the literal request.

### Plan Review (Eng Lead Mode)

```
/plan-eng-review
```

Describe your implementation plan. Get architecture diagrams, edge case maps, test matrices, and a structured walkthrough.

### Code Review

```
/review https://github.com/org/repo/pull/123
```

Get a two-pass review: critical issues (SQL safety, race conditions, trust boundaries) first, then informational issues.

### Ship a Branch

```
/ship
```

Syncs main, runs tests, bumps version, updates changelog, commits, pushes, and opens a PR. Non-interactive — just runs.

### QA Testing

```
/qa https://myapp.com
/qa --quick
/qa  ← auto-detects feature branch and tests changed pages
```

Requires gstack browser binary — see Setup below.

### Engineering Retro

```
/retro
/retro 14d
/retro compare
```

Analyzes commit history for the time window. Team-aware: per-person praise and growth areas.

## Setup

### Conversational skills (no setup required)

`/plan-ceo-review`, `/plan-eng-review`, `/review`, `/ship`, and `/retro` work immediately after installing the plugin.

### Browser skills (one-time build)

`/browse`, `/qa`, and `/setup-browser-cookies` require the gstack browser binary:

```bash
git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup
```

Requirements: [Bun](https://bun.sh/) v1.0+, macOS or Linux (x64 or arm64).

The binary is ~58MB and compiles in ~10 seconds. After setup, the browser persists between sessions with cookies, tabs, and login state intact.

## Standalone + Supercharged

Every command works without integrations:

| Command | Standalone | Supercharged With |
|---------|------------|-------------------|
| `/plan-ceo-review` | Describe your idea | Knowledge base (docs, prior ADRs) |
| `/plan-eng-review` | Describe your plan | Knowledge base, source control |
| `/review` | Paste diff or PR URL | GitHub (pull PR automatically), Greptile (triage bot comments) |
| `/ship` | Describe your branch | GitHub (create PR, post comments), Greptile (triage bot comments) |
| `/browse` | Provide URL directly | — (browser runs locally) |
| `/qa` | Provide URL | GitHub (analyze PR diff), browser |
| `/retro` | Describe your week | GitHub (commit history, PRs) |

## MCP Integrations

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](CONNECTORS.md).

| Category | Examples | What It Enables |
|---|---|---|
| **Source control** | GitHub | PR diffs, commit history, create PRs, post comments |
| **Code intelligence** | Greptile | AI code review on every PR; `/review` and `/ship` triage bot comments (VALID / ALREADY-FIXED / FALSE-POSITIVE) |
| **Project tracker** | Linear | Link review findings to tickets |
| **Knowledge base** | Notion | Prior ADRs, runbooks, architecture docs |
| **Chat** | Slack | Notify team of retro results, share QA reports |

See [CONNECTORS.md](CONNECTORS.md) for the full list of supported integrations.

## About

Created by [Garry Tan](https://x.com/garrytan), President & CEO of [Y Combinator](https://www.ycombinator.com/).

Source: [github.com/garrytan/gstack](https://github.com/garrytan/gstack)
