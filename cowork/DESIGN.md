# Software Design Document — Claude Cowork Plugin (gstack)

**Document ID:** gstack-cowork-SDD-1.0  
**Version:** 1.0  
**Date:** 2026-03-14  
**Author:** Garry Tan / gstack contributors  
**Status:** Approved  

---

## Table of Contents

1. [Introduction](#1-introduction)
2. [System Context](#2-system-context)
3. [Architecture Overview](#3-architecture-overview)
4. [Plugin Manifest & MCP Configuration](#4-plugin-manifest--mcp-configuration)
5. [Connector Architecture](#5-connector-architecture)
6. [Skill Catalog](#6-skill-catalog)
7. [Skill Design — `/plan-ceo-review`](#7-skill-design----plan-ceo-review)
8. [Skill Design — `/plan-eng-review`](#8-skill-design----plan-eng-review)
9. [Skill Design — `/review`](#9-skill-design----review)
10. [Skill Design — `/ship`](#10-skill-design----ship)
11. [Skill Design — `/browse`](#11-skill-design----browse)
12. [Skill Design — `/qa`](#12-skill-design----qa)
13. [Skill Design — `/setup-browser-cookies`](#13-skill-design----setup-browser-cookies)
14. [Skill Design — `/retro`](#14-skill-design----retro)
15. [Cross-Cutting Concerns](#15-cross-cutting-concerns)
16. [Data Stores](#16-data-stores)
17. [Deployment & Installation](#17-deployment--installation)
18. [Quality Attributes](#18-quality-attributes)
19. [Constraints & Decisions](#19-constraints--decisions)
20. [Glossary](#20-glossary)

---

## 1. Introduction

### 1.1 Purpose

This Software Design Document describes the architecture, component structure, workflow design, and integration model of the **gstack Cowork Plugin** — an opinionated engineering workflow plugin for [Claude Cowork](https://claude.ai/product/cowork) that turns Claude into a roster of on-demand engineering specialists.

### 1.2 Scope

This document covers:

- Plugin manifest and MCP server configuration (`cowork/.mcp.json`, `cowork/.claude-plugin/plugin.json`)
- All eight skill definitions (`cowork/skills/*/SKILL.md`)
- Connector architecture (`cowork/CONNECTORS.md`)
- Standalone vs. supercharged operating modes
- Greptile bot integration (implemented in `/review` and `/ship`)
- Browser binary dependency model

Out of scope: the gstack browser binary implementation (`browse/src/`), the Claude Code variant of gstack, and the gstack CI/build system.

### 1.3 Audience

- Engineers extending or maintaining the gstack Cowork plugin
- Engineering managers evaluating the plugin for team adoption
- Reviewers auditing the design for completeness and correctness

### 1.4 Definitions

See [Section 20 — Glossary](#20-glossary) for all terms.

---

## 2. System Context

### 2.1 Context Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                         USER                                          │
│  (engineer, EM, founder in Claude Cowork conversation window)        │
└────────────────────────────┬─────────────────────────────────────────┘
                             │  slash command / natural language
                             ▼
┌──────────────────────────────────────────────────────────────────────┐
│                   CLAUDE COWORK RUNTIME                               │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │               gstack Plugin                                   │   │
│  │  plugin.json  ·  .mcp.json  ·  CONNECTORS.md                 │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐        │   │
│  │  │/plan-ceo │ │/plan-eng │ │ /review  │ │  /ship   │        │   │
│  │  │-review   │ │-review   │ │          │ │          │        │   │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘        │   │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────────────────┐         │   │
│  │  │ /browse  │ │   /qa    │ │  /setup-browser-      │         │   │
│  │  │          │ │          │ │  cookies              │         │   │
│  │  └──────────┘ └──────────┘ └──────────────────────┘         │   │
│  │  ┌──────────┐                                                 │   │
│  │  │  /retro  │                                                 │   │
│  │  └──────────┘                                                 │   │
│  └──────────────────────────────────────────────────────────────┘   │
└──────┬──────────────┬───────────────────────────────────────────────┘
       │              │
       │ MCP (HTTP)   │ local binary
       │              │
┌──────▼───────┐  ┌───▼──────────────────────────────────────────────┐
│  External    │  │  LOCAL HOST (user's machine)                      │
│  MCP Servers │  │  ┌────────────────────────────────────────────┐  │
│  ─────────── │  │  │  gstack browser binary                     │  │
│  github      │  │  │  ~/.claude/skills/gstack/browse/dist/browse│  │
│  gitlab      │  │  │  (headless Chromium via Playwright/Bun)    │  │
│  linear      │  │  └────────────────────────────────────────────┘  │
│  notion      │  │  ┌────────────────────────────────────────────┐  │
│  slack       │  │  │  ~/.gstack/                                │  │
│              │  │  │    greptile-history.md                     │  │
└──────────────┘  │  └────────────────────────────────────────────┘  │
                  └──────────────────────────────────────────────────┘
```

### 2.2 Operating Modes

Every skill supports two operating modes. Skills degrade gracefully — they never fail when a connector is absent.

| Mode | Condition | Capability |
|------|-----------|------------|
| **Standalone** | No MCP connectors attached | Works from pasted diffs, described plans, or provided URLs |
| **Supercharged** | One or more MCP connectors active | Pulls data automatically (PR diffs, commit history, knowledge base docs, etc.) |

---

## 3. Architecture Overview

### 3.1 Component Hierarchy

```
cowork/
├── .claude-plugin/
│   └── plugin.json              ← Plugin metadata (name, version, author)
├── .mcp.json                    ← MCP server declarations (5 servers)
├── CONNECTORS.md                ← Human-readable connector guide
├── README.md                    ← Plugin overview and installation
├── DESIGN.md                    ← THIS DOCUMENT
└── skills/
    ├── plan-ceo-review/
    │   └── SKILL.md             ← CEO/founder plan review
    ├── plan-eng-review/
    │   └── SKILL.md             ← Engineering lead plan review
    ├── review/
    │   └── SKILL.md             ← Pre-landing code review + Greptile
    ├── ship/
    │   └── SKILL.md             ← Fully automated ship workflow
    ├── browse/
    │   └── SKILL.md             ← Headless browser automation
    ├── qa/
    │   └── SKILL.md             ← Systematic QA testing
    ├── setup-browser-cookies/
    │   └── SKILL.md             ← Cookie import for authenticated testing
    └── retro/
        └── SKILL.md             ← Engineering retrospective
```

### 3.2 Skill Execution Model

Each skill is a SKILL.md file that Claude Cowork loads as a context document. When a user invokes a slash command (e.g., `/ship`) or triggers a matching natural-language phrase, Claude executes the instructions in the corresponding SKILL.md.

```
User invokes /ship
       │
       ▼
Cowork runtime matches trigger → loads cowork/skills/ship/SKILL.md
       │
       ▼
Claude reads SKILL.md as its operating instructions
       │
       ▼
Claude executes steps in order, calling tools (bash, MCP servers,
browser binary) and asking AskUserQuestion only at designated stops
       │
       ▼
Final output: PR URL (or report, or review output)
```

### 3.3 Skill Classification

| Class | Skills | External Dependencies |
|-------|--------|-----------------------|
| **Conversational** (no binary) | `/plan-ceo-review`, `/plan-eng-review`, `/review`, `/ship`, `/retro` | MCP servers (optional) |
| **Browser-dependent** | `/browse`, `/qa`, `/setup-browser-cookies` | gstack browser binary (required) |

---

## 4. Plugin Manifest & MCP Configuration

### 4.1 Plugin Manifest (`cowork/.claude-plugin/plugin.json`)

```
Name:        gstack
Version:     1.1.0
Description: Opinionated engineering workflow skills for Claude Cowork
Author:      Garry Tan (github.com/garrytan/gstack)
```

### 4.2 MCP Server Configuration (`cowork/.mcp.json`)

All five MCP servers use the HTTP transport type. Each is optional — skills detect availability at runtime and degrade gracefully.

| Key | URL | Skills That Use It |
|-----|-----|--------------------|
| `github` | `https://api.github.com/mcp` | `/review`, `/ship`, `/qa`, `/plan-ceo-review`, `/plan-eng-review`, `/retro` |
| `gitlab` | `https://gitlab.com/api/mcp` | `/review`, `/plan-eng-review`, `/plan-ceo-review`, `/retro` (MR diffs, commit history) |
| `linear` | `https://mcp.linear.app/mcp` | `/review`, `/ship` |
| `notion` | `https://mcp.notion.com/mcp` | `/plan-ceo-review`, `/plan-eng-review`, `/review` |
| `slack` | `https://mcp.slack.com/mcp` | `/retro`, `/qa` |

**Configuration format (each entry):**
```json
"<key>": {
  "type": "http",
  "url": "<endpoint>"
}
```

---

## 5. Connector Architecture

### 5.1 Connector Matrix

| Connector | Category | Skills Supercharged |
|-----------|----------|---------------------|
| **GitHub** | Source Control | Pull PR/MR diffs automatically; read commit history; create PRs; post review comments; read Greptile bot comments |
| **GitLab** | Source Control | Pull MR diffs; read commit history |
| **Linear** | Project Tracker | Link review findings to tickets; create follow-up issues |
| **Jira / Atlassian** | Project Tracker | Same as Linear (via Linear MCP or direct) |
| **Asana** | Project Tracker | Same as Linear |
| **Notion** | Knowledge Base | Read prior ADRs, architecture docs, coding standards |
| **Confluence** | Knowledge Base | Same as Notion |
| **Slack** | Communication | Post retro summaries; notify on QA completion |

### 5.2 Connector Detection Pattern

Each skill checks connector availability inline before using it:

```
If <connector> is connected:
  → Use MCP tool to fetch data automatically
Else (standalone):
  → Ask user to paste or describe the data, OR
  → Proceed with locally available data (git, files, etc.)
```

This pattern is used in every skill's SKILL.md. Skills never hard-fail on a missing connector.

### 5.3 Connector Installation

```
Cowork → Customize → Plugins → gstack → Connectors
  → Toggle each connector ON
  → Authorize the OAuth integration
```

---

## 6. Skill Catalog

### 6.1 Skill Summary Table

| Skill | Slash Command | Trigger Phrases | Input | Output | Binary? |
|-------|---------------|-----------------|-------|--------|---------|
| CEO Plan Review | `/plan-ceo-review` | "founder review", "10-star product", "am I building the right thing" | Plan description / PR / issue | Mode-selected review with diagrams | No |
| Eng Plan Review | `/plan-eng-review` | "engineering review", "architect this", "technical review" | Plan description / PR / issue | Structured review: architecture, code quality, tests, performance | No |
| Code Review | `/review` | "code review", "review this PR", "review before merge", "is this code safe" | PR URL / diff / file path | Two-pass review report + Greptile triage | No |
| Ship | `/ship` | "ship", "ship this", "land this branch", "create PR", "push and PR" | (runs on current branch) | PR URL | No |
| Browse | `/browse` | "browse", "navigate to", "take a screenshot", "test this URL" | URL / browser command | Command output, screenshots | **Yes** |
| QA | `/qa` | "qa", "test this site", "find bugs", "smoke test" | URL / mode flags | QA report with health score | **Yes** |
| Cookie Setup | `/setup-browser-cookies` | "import cookies", "set up browser cookies", "authenticate the browser" | Browser name / domain | Imported cookies confirmed | **Yes** |
| Retro | `/retro` | "retro", "retrospective", "weekly review", "team review" | Time window argument | Full retro report per-person | No |

### 6.2 Standalone vs. Supercharged — Full Matrix

| Skill | Standalone Capability | Supercharged With | Additional Capability |
|-------|-----------------------|-------------------|-----------------------|
| `/plan-ceo-review` | Describe your plan | GitHub | Repo system audit, open PRs, file state |
| | | Notion/Confluence | Prior ADRs, team standards |
| `/plan-eng-review` | Describe your plan | GitHub | PR/issue diff, related files, conflict detection |
| | | Notion/Confluence | ADRs, architecture docs |
| `/review` | Paste diff or point to files | GitHub | Auto-pull PR diff, CI status, post comments, Greptile triage |
| | | Notion/Confluence | Team coding standards |
| `/ship` | Describe branch (git available locally) | GitHub | Create PR via API, post comments |
| | | GitHub | Greptile bot comment triage (Step 3.75) |
| | | Linear/Jira | Link PR to ticket |
| `/browse` | URL provided directly | — | (browser runs locally; no MCP enhancement) |
| `/qa` | URL provided | GitHub | Auto-pull PR diff for diff-aware mode; post QA report as PR comment |
| `/setup-browser-cookies` | — | — | (always requires binary; no MCP enhancement) |
| `/retro` | Describe your week | GitHub | Auto-pull commit history, PRs, review activity |
| | | Slack | Post retro summary to team channel |

---

## 7. Skill Design — `/plan-ceo-review`

**File:** `cowork/skills/plan-ceo-review/SKILL.md`

### 7.1 Purpose

Founder/CEO-mode plan review. Challenges premises, stress-tests failure modes, demands observability, and ensures the plan is the most direct path to the desired outcome — not a proxy solution. Does NOT write code.

### 7.2 Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│  PRE-REVIEW SYSTEM AUDIT (if GitHub connected)                       │
│  • Last 30 commits on main                                           │
│  • Open PRs and in-flight branches                                   │
│  • README / ARCHITECTURE / CLAUDE.md                                 │
├─────────────────────────────────────────────────────────────────────┤
│  STEP 0: Nuclear Scope Challenge + Mode Selection                    │
│  0A. Premise Challenge (right problem? right path?)                  │
│  0B. Existing Code Leverage (what already exists?)                   │
│  0C. Dream State Mapping (current → this plan → 12-month ideal)      │
│  0D. Mode-Specific Analysis                                          │
│  0E. Mode Selection → ASK USER (STOP until answered)                │
├─────────────────────────────────────────────────────────────────────┤
│  REVIEW SECTIONS (after mode agreed)                                 │
│  1. Architecture Review (with required ASCII diagram)               │
│  2. Error & Rescue Map (method × failure × exception table)          │
│  3. Security & Threat Model                                          │
│  4. Test Coverage Matrix                                             │
│  5. Observability (logs, metrics, alerts, runbook)                   │
│  6. Deployment Plan (flags, migrations, rollback, canary)            │
├─────────────────────────────────────────────────────────────────────┤
│  FINAL OUTPUT                                                        │
│  • Summary paragraph                                                 │
│  • Open questions list                                               │
│  • Recommended next step                                             │
└─────────────────────────────────────────────────────────────────────┘
```

### 7.3 Three Modes

| Mode | Posture | Default Context |
|------|---------|-----------------|
| **SCOPE EXPANSION** | Build the cathedral. Push scope up 10x. | Greenfield features |
| **HOLD SCOPE** | Plan is right. Make it bulletproof. | Bug fixes, hotfixes |
| **SCOPE REDUCTION** | Surgeon. Strip to minimum viable. | Plans touching >15 files |

**Mode commitment rule:** Concerns raised once in Step 0. After mode is selected, commit fully. No lobbying.

### 7.4 Prime Directives (enforced throughout)

1. Zero silent failures
2. Every error has a name
3. Data flows have shadow paths (nil, empty, error)
4. Interactions have edge cases (double-click, stale state, back button)
5. Observability is scope, not afterthought
6. Diagrams are mandatory (ASCII art)
7. Everything deferred must be in TODOS.md
8. Optimize for 6-month future
9. Permission to say "scrap it and do this instead"

### 7.5 Priority Hierarchy Under Context Pressure

```
Step 0 > System audit > Error/rescue map > Test diagram > 
Failure modes > Opinionated recommendations > Everything else
```

### 7.6 Connector Enhancements

| Connector | Enhancement |
|-----------|-------------|
| GitHub | System audit (recent commits, open PRs, README/ARCHITECTURE) |
| Notion/Confluence | Prior ADRs, team coding standards |

---

## 8. Skill Design — `/plan-eng-review`

**File:** `cowork/skills/plan-eng-review/SKILL.md`

### 8.1 Purpose

Engineering lead-mode plan review. Validates technical execution — architecture, data flow, edge cases, test coverage, performance — before any code is written. Interactive: one issue per question, one section at a time.

### 8.2 Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│  STEP 0: Scope Challenge                                             │
│  • What existing code already partially solves this?                │
│  • Minimum set of changes for the goal?                             │
│  • Complexity check: >8 files or >2 new classes → smell            │
│  → Present 3 options: SCOPE REDUCTION / BIG CHANGE / SMALL CHANGE  │
│  → STOP. Wait for user selection.                                   │
├─────────────────────────────────────────────────────────────────────┤
│  Section 1: Architecture Review                              STOP   │
│  • System design, component boundaries, data flow                   │
│  • Dependency graph, scaling, SPOF, security                        │
│  • ASCII diagram for every new data flow / state machine            │
│  • Production failure scenarios                                     │
├─────────────────────────────────────────────────────────────────────┤
│  Section 2: Code Quality Review                             STOP    │
│  • DRY violations (aggressive), error handling, tech debt           │
│  • Over/under-engineering check, naming clarity                     │
├─────────────────────────────────────────────────────────────────────┤
│  Section 3: Test Coverage                                   STOP    │
│  • Test matrix (SCENARIO × UNIT/INTEGRATION/E2E)                   │
│  • Flag every uncovered scenario                                    │
├─────────────────────────────────────────────────────────────────────┤
│  Section 4: Performance Review                              STOP    │
│  • N+1 queries, missing indexes, O(n²) hot paths                    │
│  • Memory, async boundaries, caching                                │
├─────────────────────────────────────────────────────────────────────┤
│  COMPLETION SUMMARY                                                  │
│  ## Plan Review Complete                                             │
│  Sections / Issues found & resolved / Open questions /              │
│  Recommended first commit                                            │
└─────────────────────────────────────────────────────────────────────┘
```

### 8.3 Interaction Protocol

- **One issue per AskUserQuestion.** Never batch.
- Present options, state recommendation, explain WHY.
- Do not proceed past a section until all issues in that section are resolved.
- If context pressure: compress to Step 0 + combined pass, one issue per section, single question round.

### 8.4 Engineering Preferences Enforced

- DRY: flag repetition aggressively
- Well-tested code is non-negotiable
- "Engineered enough": not fragile, not over-abstracted
- Explicit over clever
- Minimal diff
- Stale ASCII diagrams are worse than no diagrams

### 8.5 Test Matrix Template

```
SCENARIO               | UNIT | INTEGRATION | E2E
-----------------------|------|-------------|-----
Happy path             |  ✓   |      ✓      |  ✓
Empty/nil input        |  ✓   |      —      |  —
Auth failure           |  ✓   |      ✓      |  ✓
Concurrent writes      |  ✓   |      ✓      |  —
Rate limit / timeout   |  ✓   |      —      |  —
Malformed API response |  ✓   |      —      |  —
```

### 8.6 Connector Enhancements

| Connector | Enhancement |
|-----------|-------------|
| GitHub | Read PR/issue/branch, pull related files, check for conflicting open PRs |
| Notion/Confluence | Prior ADRs, team coding standards |

---

## 9. Skill Design — `/review`

**File:** `cowork/skills/review/SKILL.md`

### 9.1 Purpose

Paranoid pre-landing code review. Finds bugs that pass CI but blow up in production. Two-pass: critical blockers first, informational findings second. Includes Greptile bot comment triage when GitHub is connected.

### 9.2 Workflow

```
┌─────────────────────────────────────────────────────────────────────┐
│  Step 1: Check Branch / Get Diff                                     │
│  • GitHub connected + PR URL → pull diff automatically              │
│  • Standalone → use pasted diff or ask for context                  │
│  • On main with no changes → output notice and STOP                 │
├─────────────────────────────────────────────────────────────────────┤
│  Step 2: Two-Pass Review                                             │
│  Pass 1 — CRITICAL:                                                  │
│    · SQL & Data Safety (injection, TOCTOU, validation bypass, N+1)  │
│    · Race Conditions & Concurrency (read-check-write, XSS)          │
│    · LLM Output Trust Boundary (unvalidated LLM → DB/mailer)        │
│  Pass 2 — INFORMATIONAL:                                             │
│    · Conditional Side Effects                                        │
│    · Magic Numbers & String Coupling                                 │
│    · Dead Code & Consistency                                         │
│    · LLM Prompt Issues                                               │
│    · Test Gaps                                                       │
│    · View / Frontend                                                 │
├─────────────────────────────────────────────────────────────────────┤
│  Step 3: Output Findings                                             │
│  • CRITICAL found → for each: AskUserQuestion (Fix/Acknowledge/FP)  │
│  • Only informational → output findings, no action needed           │
│  • Nothing found → "Pre-Landing Review: No issues found."            │
├─────────────────────────────────────────────────────────────────────┤
│  Step 4: Greptile Review  [GitHub-connected only]                    │
│  • Fetch greptile-apps[bot] comments (line-level + top-level)       │
│  • Suppress known false positives from ~/.gstack/greptile-history.md│
│  • Classify: VALID / ALREADY-FIXED / FALSE-POSITIVE                 │
│  • Output "## Greptile Review: N comments (X valid, Y fixed, Z FP)" │
│  • VALID → AskUserQuestion (Fix/Acknowledge/False positive)          │
│    └─ False positive chosen → save to greptile-history.md (type: fp)│
│  • FALSE-POSITIVE → AskUserQuestion (Confirm/Fix anyway/Ignore)     │
│    └─ Confirmed → save to greptile-history.md (type: fp)            │
└─────────────────────────────────────────────────────────────────────┘
```

### 9.3 Review Checklist — Pass 1 (CRITICAL)

| Category | Issues Checked |
|----------|----------------|
| **SQL & Data Safety** | String interpolation in SQL; TOCTOU check-then-set; `update_column` bypassing validation; N+1 queries |
| **Race Conditions** | Read-check-write without constraint; `find_or_create_by` without unique index; non-atomic status transitions; XSS via `html_safe`/`raw()` |
| **LLM Trust Boundary** | LLM-generated values written to DB without validation; untyped tool output accepted before DB writes |

### 9.4 Review Checklist — Pass 2 (INFORMATIONAL)

| Category | Issues Checked |
|----------|----------------|
| **Conditional Side Effects** | Branch forgets a side effect; log claims action happened when it was skipped |
| **Magic Numbers** | Bare literals in multiple files; error strings used as query filters |
| **Dead Code** | Methods never called; duplicate logic |
| **LLM Prompts** | User data in prompt without sanitization; missing output format validation |
| **Test Gaps** | Missing failure mode tests; public methods untested; mocks testing themselves |
| **View/Frontend** | Strings not in i18n; hardcoded URLs/colors; missing loading/error states |

### 9.5 Output Format

```
Pre-Landing Review: N issues (X critical, Y informational)

**CRITICAL** (blocking):
- [file:line] Problem description
  Fix: suggested fix

**Issues** (non-blocking):
- [file:line] Problem description
  Fix: suggested fix
```

### 9.6 Greptile Integration Design

```
┌──────────────────────────────────────────────────────────────────┐
│  GREPTILE TRIAGE (Step 4 of /review)                              │
├──────────────────────────────────────────────────────────────────┤
│  Precondition: GitHub connected AND PR URL provided               │
│                                                                    │
│  1. FETCH                                                          │
│     GitHub API → PR review comments (line-level, position≠null)   │
│     GitHub API → PR issue comments (top-level)                    │
│     Filter: user.login == "greptile-apps[bot]"                    │
│                                                                    │
│  2. SUPPRESS                                                       │
│     Read ~/.gstack/greptile-history.md                            │
│     Match: type==fp AND repo match AND file-pattern match AND      │
│            category match → skip as SUPPRESSED                    │
│                                                                    │
│  3. CLASSIFY (per non-suppressed comment)                          │
│     Read file @ path:line ±10 lines                               │
│     Cross-reference full diff                                      │
│     → VALID: real bug in current code                             │
│     → ALREADY-FIXED: real issue, addressed in later commit        │
│     → FALSE-POSITIVE: misunderstood code / stylistic noise        │
│                                                                    │
│  4. PRESENT                                                        │
│     Header: "## Greptile Review: N (X valid, Y fixed, Z false positives)"│
│     Per comment: [TAG] file:line — summary — <link>               │
│                                                                    │
│  5. ACT                                                            │
│     VALID → AskUserQuestion: Fix / Acknowledge / False positive    │
│              False positive chosen → append to greptile-history.md│
│     FALSE-POSITIVE → AskUserQuestion: Confirm / Fix anyway / Skip │
│              Confirmed → append to greptile-history.md            │
└──────────────────────────────────────────────────────────────────┘
```

**History file format** (`~/.gstack/greptile-history.md`):
```
<YYYY-MM-DD> | <owner/repo> | <type> | <file-pattern> | <category>
```

Types: `fp`, `fix`, `already-fixed`  
Categories: `race-condition`, `null-check`, `error-handling`, `style`, `type-safety`, `security`, `performance`, `correctness`, `other`

### 9.7 Invariants

- Read the **full diff** before commenting — never flag issues already addressed in the diff
- Read-only by default — only modify files if user explicitly chooses "Fix it now"
- Be terse: one line problem, one line fix. No preamble.
- Only flag real problems

---

## 10. Skill Design — `/ship`

**File:** `cowork/skills/ship/SKILL.md`

### 10.1 Purpose

Fully automated ship workflow. From a ready feature branch: sync main, run tests, pre-landing review, Greptile triage, bump version, generate CHANGELOG, create bisectable commits, push, open PR. Non-interactive — runs straight through.

### 10.2 Interrupt Conditions

The workflow **stops** only for:

| Condition | Action |
|-----------|--------|
| On `main` branch | Abort: "You're on main. Ship from a feature branch." |
| Unresolvable merge conflicts | Stop, show conflicts |
| Test failures | Stop, show failures |
| CRITICAL review issue where user chose Fix | Apply, commit, stop, ask to re-run `/ship` |
| Greptile VALID issue requiring user decision | Prompt user |
| MINOR or MAJOR version bump needed | Ask once |

The workflow **never stops** for:
- Uncommitted changes (always included)
- MICRO/PATCH version bump (auto-decided)
- CHANGELOG content (auto-generated)
- Commit message approval (auto-committed)

### 10.3 Workflow

```
Step 1: Pre-flight
  └─ Check branch not main
  └─ git status (uncommitted changes always included)
  └─ git diff main...HEAD --stat + git log main..HEAD --oneline

Step 2: Merge origin/main
  └─ git fetch origin main && git merge origin/main --no-edit
  └─ Auto-resolve simple conflicts (VERSION, CHANGELOG ordering)
  └─ Complex conflicts → STOP

Step 3: Run Tests
  └─ Detect test runner (npm/yarn/rspec/pytest/go test/bun)
  └─ Run all suites
  └─ Any failure → STOP, show failures

Step 3.5: Pre-Landing Review
  └─ git diff origin/main → full diff
  └─ Two-pass review (same checklist as /review)
  └─ Output: "Pre-Landing Review: N issues (X critical, Y informational)"
  └─ CRITICAL issues → AskUserQuestion per issue (Fix/Acknowledge/Skip)
     └─ User chose Fix → apply fixes, commit fixed files, STOP, ask to re-run
     └─ User chose B/C → continue

Step 3.75: Greptile Review  [GitHub-connected only]
  └─ Fetch greptile-apps[bot] comments
  └─ Suppress known false positives from ~/.gstack/greptile-history.md
  └─ Classify: VALID / ALREADY-FIXED / FALSE-POSITIVE
  └─ ALREADY-FIXED → auto-reply "Good catch — already fixed in <sha>."
     └─ Save to history (type: already-fixed)
  └─ FALSE-POSITIVE → auto-reply with explanation
     └─ Save to history (type: fp)
  └─ VALID → AskUserQuestion (Fix/Acknowledge/False positive)
     └─ Fix → apply, commit, reply "Fixed in <sha>.", save history (type: fix)
     └─ False positive chosen → reply explanation, save history (type: fp)
  └─ Any VALID fixes applied → RE-RUN Step 3 (tests)
  └─ Save results for Step 8 PR body

Step 4: Version Bump (auto-decide)
  └─ Read VERSION file
  └─ Count lines changed: git diff origin/main...HEAD --stat | tail -1
  └─ < 50 lines → MICRO (4-digit) or PATCH (semver)  [auto]
  └─ 50+ lines → PATCH (4-digit) or MINOR (semver)   [auto]
  └─ Major feature / architectural → MINOR (4-digit) [ASK]
  └─ Milestone / breaking → MAJOR                     [ASK]
  └─ Write new version to VERSION file

Step 5: CHANGELOG (auto-generate)
  └─ git log main..HEAD --oneline (ALL commits)
  └─ git diff main...HEAD (full diff)
  └─ Categorize: Added / Changed / Fixed / Removed
  └─ Insert after file header, dated today
  └─ Format: ## [X.Y.Z] - YYYY-MM-DD

Step 6: Commit (bisectable chunks)
  └─ Group changes into logical commits (1 commit = 1 coherent change)
  └─ Order: infrastructure → models+services → controllers+views → VERSION+CHANGELOG
  └─ Each commit must be independently valid
  └─ Final commit: "chore: bump version and changelog (vX.Y.Z)"

Step 7: Push
  └─ git push -u origin <branch-name>

Step 8: Create PR
  └─ GitHub connected → GitHub MCP create PR
  └─ GitHub not connected → gh pr create (local CLI)
  └─ Output: PR URL
```

### 10.4 PR Body Template

```markdown
## Summary
<bullet points from CHANGELOG>

## Pre-Landing Review
<findings from Step 3.5, or "No issues found.">

## Greptile Review
<[FIXED]/[FALSE POSITIVE]/[ALREADY-FIXED]/[VALID] tags per comment>
<or "No Greptile comments.">
<omit section entirely if GitHub was not connected>

## Test plan
- [x] All tests pass (N total, 0 failures)

🤖 Generated with Claude Code
```

### 10.5 Version Bump Decision Tree

```
git diff stat → lines changed
         │
    < 50 lines ──────────────────────── auto → MICRO / PATCH
         │
    50+ lines ──────────────────────── auto → PATCH / MINOR
         │
    Major feature ──────────────────── ASK → MINOR / MAJOR
         │
    Breaking change / milestone ──── ASK → MAJOR
```

Bumping a digit resets all lower digits to 0.  
Example: `0.19.1.0` + PATCH → `0.19.2.0`

### 10.6 Greptile Integration Design (Ship Context)

The Greptile step in `/ship` differs from `/review` in two key ways:

1. **Auto-replies** for ALREADY-FIXED and FALSE-POSITIVE (no user prompt needed — the ship workflow is non-interactive)
2. **Test re-run gate**: if any VALID Greptile issue was fixed in Step 3.75, tests must re-run before continuing to Step 4

```
                     FALSE-POSITIVE
                    ┌──────────────── auto-reply (explanation) → save history fp
                    │
Comment found ──────┤  ALREADY-FIXED
                    ├──────────────── auto-reply ("Good catch, fixed in <sha>") → save history already-fixed
                    │
                    │  VALID
                    └──────────────── AskUserQuestion
                                          ├─ Fix → apply, commit, reply "Fixed in <sha>", history fix
                                          │         └─ RE-RUN TESTS
                                          ├─ Acknowledge → continue
                                          └─ FP → reply explanation, history fp
```

---

## 11. Skill Design — `/browse`

**File:** `cowork/skills/browse/SKILL.md`

### 11.1 Purpose

Headless browser automation for QA testing, deployment verification, and dogfooding. Wraps a persistent headless Chromium instance (via the gstack browser binary) and exposes a rich command set via a compiled Bun/Playwright binary.

### 11.2 Binary Architecture

```
~/.claude/skills/gstack/
└── browse/
    └── dist/
        └── browse          ← compiled binary (~58MB, Bun + Playwright Chromium)
```

**Binary lookup order:**
```bash
_ROOT=$(git rev-parse --show-toplevel 2>/dev/null)
B=""
[ -n "$_ROOT" ] && [ -x "$_ROOT/.claude/skills/gstack/browse/dist/browse" ] && B="..."
[ -z "$B" ] && B=~/.claude/skills/gstack/browse/dist/browse
```

**Lifecycle:**
- First call: auto-starts (~3 second cold start)
- Subsequent calls: ~100–200ms per command
- Auto-shutdown after 30 minutes idle
- State persists between calls (cookies, tabs, login sessions)

### 11.3 Command Reference (grouped)

| Group | Commands | Description |
|-------|----------|-------------|
| **Navigation** | `goto`, `back`, `forward`, `reload`, `url` | Page navigation |
| **Reading** | `text`, `html`, `links`, `forms`, `accessibility` | Content extraction |
| **Interaction** | `click`, `fill`, `type`, `hover`, `press`, `select`, `upload`, `scroll`, `wait` | User interaction simulation |
| **Browser Control** | `cookie`, `cookie-import`, `cookie-import-browser`, `header`, `useragent`, `viewport`, `dialog-*` | Session/environment control |
| **Inspection** | `snapshot`, `screenshot`, `console`, `network`, `js`, `eval`, `is`, `attrs`, `css`, `cookies`, `storage`, `perf`, `dialog` | State inspection |
| **Visual** | `responsive`, `diff`, `pdf` | Layout and comparison |
| **Tabs** | `tabs`, `newtab`, `tab`, `closetab` | Multi-tab management |
| **Server** | `status`, `restart`, `stop` | Binary lifecycle |
| **Meta** | `chain` | Batch command execution |

### 11.4 Snapshot System

The snapshot is the primary tool for understanding pages and targeting interactions.

```
$B snapshot [flags]

Flags:
  -i   Interactive elements only (buttons, links, inputs) → @e refs
  -c   Compact (no empty structural nodes)
  -d N Limit tree depth
  -s   Scope to CSS selector
  -D   Unified diff against previous snapshot
  -a   Annotated screenshot (red overlay boxes + ref labels)
  -o   Output path for annotated screenshot
  -C   Cursor-interactive elements (@c refs) — finds clickable divs
```

After snapshot, `@e1`, `@e2`, ... refs are usable as selectors in any command:
```bash
$B click @e3       # click the 3rd interactive element
$B fill @e4 "val"  # fill by ref
```

### 11.5 Typical QA Flow Pattern

```
$B goto <url>                         # 1. Navigate
$B snapshot -i                        # 2. Map interactive elements
$B fill @e3 "input"                   # 3. Interact by ref
$B click @e5
$B snapshot -D                        # 4. Diff to verify change
$B is visible ".success"              # 5. Assert state
$B console                            # 6. Check for JS errors
$B screenshot /tmp/evidence.png       # 7. Capture evidence
```

### 11.6 Browser Persistence Model

The binary maintains a persistent Chromium process. All browser state (cookies, localStorage, sessionStorage, tabs, active sessions) carries over between Claude conversations as long as the idle timeout has not elapsed.

---

## 12. Skill Design — `/qa`

**File:** `cowork/skills/qa/SKILL.md`

### 12.1 Purpose

Systematic QA testing of web applications. Four modes covering diff-aware branch verification, full exploration, quick smoke testing, and regression comparison. Produces a structured report with health score, screenshot evidence, and repro steps.

### 12.2 Modes

| Mode | Trigger | Scope | Duration |
|------|---------|-------|----------|
| **Diff-aware** | On feature branch, no URL | Pages changed by the branch diff | ~5 min |
| **Full** | URL provided | All reachable pages | 5–15 min |
| **Quick** | `--quick` | Homepage + top 5 navigation targets | 30 seconds |
| **Regression** | `--regression <baseline>` | Full mode + diff against previous run baseline | ~15 min |

### 12.3 Workflow

```
Phase 1: Initialize
  └─ Find binary (see /browse Setup)
  └─ mkdir -p .gstack/qa-reports/screenshots
  └─ Note start time

Phase 2: Authenticate (if needed)
  └─ Fill login form via snapshot refs
  └─ OR import cookie file via $B cookie-import
  └─ Handle 2FA prompts

Phase 3: Orient
  └─ $B goto <target>
  └─ $B snapshot -i -a -o .gstack/qa-reports/screenshots/initial.png
  └─ $B links → map navigation
  └─ $B console --errors → check for landing errors
  └─ Detect framework (Next.js / Rails / WordPress / SPA)

Phase 4: Explore (per page)
  └─ $B goto <page>
  └─ $B snapshot -i -a -o <screenshot>
  └─ $B console --errors
  └─ Per-page checklist:
     1. Visual scan (layout, overflow, missing content)
     2. Interactive elements (click buttons, links, controls)
     3. Forms (fill, submit, empty/invalid/edge cases)
     4. Navigation (all paths in and out)
     5. States (empty, loading, error, overflow)
     6. Console (new JS errors after interactions)

Phase 5: Document Issues
  └─ Screenshot + console errors + repro steps + severity
  └─ Taxonomy: Visual / Functional / UX / Content / Performance / Console / Accessibility

Phase 6: Wrap Up
  └─ Health score calculation
  └─ Final report in standardized format
```

### 12.4 Diff-Aware Mode — Affected Page Detection

```
git diff main...HEAD --name-only
           │
           ▼
  Controller/route file → URL paths it serves
  View/template/component → pages that render it
  Model/service file → pages that use those models
  CSS file → pages that include the stylesheet
  API endpoint → test directly with JS fetch
           │
           ▼
  Detect running app at :3000, :4000, :8080
           │
           ▼
  Test each affected page
```

### 12.5 Health Score Dimensions

| Dimension | Max Points |
|-----------|------------|
| Visual integrity | 25 |
| Functional correctness | 30 |
| Console cleanliness | 15 |
| Navigation | 10 |
| Performance | 10 |
| Content quality | 10 |
| **Total** | **100** |

### 12.6 Issue Report Format

```markdown
# QA Report — [App Name]
**Date:** ... | **Mode:** ... | **Duration:** ... | **Health Score:** N/100

## Summary
[2-3 sentence executive summary]

## Issues Found (N total)

### Critical (N)
#### [Issue Title]
**Severity:** Critical | **Page:** [URL]
[Description]
**Steps to reproduce:** 1. ... 2. ...
**Expected:** [behavior]  **Actual:** [behavior]
**Evidence:** [screenshot path]
```

### 12.7 Connector Enhancements

| Connector | Enhancement |
|-----------|-------------|
| GitHub | Auto-pull PR diff for diff-aware mode; post QA report as PR comment; link issues to PR |

---

## 13. Skill Design — `/setup-browser-cookies`

**File:** `cowork/skills/setup-browser-cookies/SKILL.md`

### 13.1 Purpose

Import encrypted cookies from the user's real browser (Comet, Chrome, Arc, Brave, Edge) into the gstack headless Chromium session. Enables authenticated testing of private pages without manual login in the headless browser.

### 13.2 How It Works

The `cookie-import-browser` binary command reads encrypted cookies directly from the real browser's SQLite database on disk and injects them into the headless session.

```
Real Browser
  SQLite cookie DB (encrypted)
         │
         ▼ cookie-import-browser command
  gstack binary decrypts and imports
         │
         ▼
  Headless Chromium session
  (authenticated state)
```

### 13.3 Supported Browsers

| Browser | Platform Support |
|---------|-----------------|
| Comet (Anthropic) | macOS |
| Chrome | macOS, Linux |
| Arc | macOS |
| Brave | macOS, Linux |
| Edge | macOS, Linux |

### 13.4 Usage Patterns

| Mode | Command | Use Case |
|------|---------|----------|
| Interactive picker | `$B cookie-import-browser` | First-time setup; browse and select cookies via web UI |
| Direct import | `$B cookie-import-browser <browser> --domain <domain>` | Scripted / repeatable import |

### 13.5 Domain Matching

- Exact domain: `myapp.com` → matches only `myapp.com`
- Subdomain wildcard: `.myapp.com` → matches all subdomains

### 13.6 Important Behavioral Notes

- Session cookies are included in the import
- Running the same domain again replaces previous cookies
- Cookies expire — re-run the skill when authenticated sessions expire
- No MCP connector enhancement — purely local binary operation

---

## 14. Skill Design — `/retro`

**File:** `cowork/skills/retro/SKILL.md`

### 14.1 Purpose

Weekly engineering retrospective. Analyzes commit history, work patterns, and code quality metrics. Team-aware — per-person praise and growth opportunities for every contributor. Tracks trends over time. Optionally posts to Slack.

### 14.2 Workflow

```
Step 1: Gather Raw Data
  └─ GitHub connected → pull commits, PRs, review activity
  └─ Local git → git log, shortlog, numstat
  └─ Standalone → ask user to describe the week

Step 2: Compute Metrics
  └─ Commits to main, contributors, PRs merged
  └─ Total insertions/deletions, net LOC, active days
  └─ Per-author leaderboard (sort by commits, current user first labeled "You")

Step 3: Commit Time Distribution
  └─ Hourly histogram (ASCII bar chart)
  └─ Peak hours, dead zones, late-night clusters, bimodal pattern

Step 4: Work Session Detection
  └─ 45-minute gap threshold between commits
  └─ Classify: Deep (50+ min), Medium (20-50 min), Micro (<20 min)
  └─ Total active time, avg session length, LOC/hour

Step 5: Commit Type Breakdown
  └─ Conventional prefix breakdown (feat/fix/refactor/test/chore/docs)
  └─ Flag fix ratio >50% (signals "ship fast, fix fast")

Step 6: Hotspot Analysis
  └─ Top 10 most-changed files
  └─ Flag churn hotspots (5+ changes), test vs. production ratio

Step 7: Focus Score + Ship of the Week
  └─ Focus score: % of commits in single top-level directory
  └─ Ship of the week: highest-LOC PR/commit cluster

Step 8: Per-Person Deep Dive
  └─ For each contributor: Wins / Growth opportunity / Pattern
  └─ Tone: generous praise, specific actionable growth

Step 9: Compare Mode (if "compare" argument)
  └─ Run same analysis for prior same-length period
  └─ Side-by-side delta table with ✓ improvements, ⚠ regressions
```

### 14.3 Time Window Arguments

| Argument | Window |
|----------|--------|
| (none) | Last 7 days (default) |
| `24h` | Last 24 hours |
| `14d` | Last 14 days |
| `30d` | Last 30 days |
| `compare` | Current 7d vs. prior 7d |
| `compare 14d` | Current 14d vs. prior 14d |

### 14.4 Per-Person Output Template

```
### [Name] — [N commits, +X/-Y LOC]

**Wins this week:**
- [Specific achievement with evidence]

**Growth opportunity:**
- [One specific, actionable observation]

**Pattern:**
- [Work style observation]
```

**Tone guidelines:** Generous praise. Growth = systemic patterns, not individual mistakes. Frame as "how do we make next week better."

### 14.5 Connector Enhancements

| Connector | Enhancement |
|-----------|-------------|
| GitHub | Auto-pull commit history, PR titles, review counts, merge times, PR size distribution |
| Slack | Post retro summary to team channel |

---

## 15. Cross-Cutting Concerns

### 15.1 CONNECTORS.md Discovery

Every skill begins with:
```
> If you see unfamiliar placeholders or need to check which tools are
> connected, see CONNECTORS.md.
```

Skills reference `../../CONNECTORS.md` (relative to `cowork/skills/<name>/SKILL.md`). This provides a single authoritative source for connector documentation without duplicating it in every skill.

### 15.2 Graceful Degradation Pattern

Every skill follows this pattern for each external dependency:

```
IF <connector/binary> available:
  → use it, unlock supercharged capability
ELSE:
  → proceed in standalone mode
  → ask user to provide the data manually if needed
  → NEVER hard-fail on a missing connector
```

### 15.3 Greptile History Store

The `~/.gstack/greptile-history.md` file is a shared cross-skill persistent store. Both `/review` (Step 4) and `/ship` (Step 3.75) read and write to it.

**Schema:**
```
<YYYY-MM-DD> | <owner/repo> | <type> | <file-pattern> | <category>
```

**Read behavior:** parse all lines; skip lines that can't be parsed; never fail on a malformed file.  
**Write behavior:** always `mkdir -p ~/.gstack` before appending.

### 15.4 Interactivity Discipline

Skills minimize interrupts. The interactivity model per skill:

| Skill | Stops For | Never Stops For |
|-------|-----------|-----------------|
| `/plan-ceo-review` | Mode selection (Step 0), one issue per section | — |
| `/plan-eng-review` | One issue per section per stop | — |
| `/review` | Each CRITICAL issue (Fix/Acknowledge/FP), Greptile VALID/FP | Informational findings |
| `/ship` | Critical issues, MINOR/MAJOR bump, Greptile VALID | Everything else |
| `/browse` | — (pass-through to binary) | — |
| `/qa` | 2FA prompt if needed | — |
| `/setup-browser-cookies` | — | — |
| `/retro` | — | — |

### 15.5 Output Terseness Standard

The `/review` and `/ship` pre-landing review adhere to strict output constraints:
- One line describing the problem
- One line with the fix
- No preamble, no summaries, no "looks good overall"

---

## 16. Data Stores

### 16.1 Persistent Data

| Store | Path | Written By | Read By | Format |
|-------|------|------------|---------|--------|
| Greptile history | `~/.gstack/greptile-history.md` | `/review` (Step 4), `/ship` (Step 3.75) | `/review`, `/ship` | Pipe-delimited text |
| QA reports | `.gstack/qa-reports/` | `/qa` | User / GitHub PR comments | Markdown + PNG |
| QA screenshots | `.gstack/qa-reports/screenshots/` | `/qa` | User | PNG |
| Browser state | Headless Chromium process | `/browse`, `/qa`, `/setup-browser-cookies` | `/browse`, `/qa` | In-memory (Chromium) |

### 16.2 Ephemeral Data

| Store | Path | Purpose |
|-------|------|---------|
| Greptile line comments | `/tmp/greptile_line.json` | Intermediate Greptile fetch result |
| Greptile top comments | `/tmp/greptile_top.json` | Intermediate Greptile fetch result |
| Ship test output | `/tmp/ship_tests.txt` | Test run log |

### 16.3 Greptile History Record Examples

```
2026-03-13 | garrytan/myapp | fp | app/services/auth_service.rb | race-condition
2026-03-13 | garrytan/myapp | fix | app/models/user.rb | null-check
2026-03-13 | garrytan/myapp | already-fixed | lib/payments.rb | error-handling
```

---

## 17. Deployment & Installation

### 17.1 Installation Methods

**Method 1 — Cowork plugin registry:**
```bash
claude plugins add garrytan/gstack/cowork
```

**Method 2 — Manual (copy-paste prompt):**
```
Install the gstack plugin: download https://github.com/garrytan/gstack
and add the cowork/ directory as a local plugin.
```

### 17.2 Post-Install Readiness

| Skill | Ready After Install? | Additional Setup |
|-------|---------------------|------------------|
| `/plan-ceo-review` | ✓ Yes | — |
| `/plan-eng-review` | ✓ Yes | — |
| `/review` | ✓ Yes | — |
| `/ship` | ✓ Yes | — |
| `/browse` | ✗ No | Build browser binary (see below) |
| `/qa` | ✗ No | Build browser binary (see below) |
| `/setup-browser-cookies` | ✗ No | Build browser binary (see below) |
| `/retro` | ✓ Yes | — |

### 17.3 Browser Binary Build (One-Time)

```bash
git clone https://github.com/garrytan/gstack.git ~/.claude/skills/gstack
cd ~/.claude/skills/gstack && ./setup
```

**Requirements:** Bun v1.0+, macOS or Linux (x64 or arm64)  
**Output:** `~/.claude/skills/gstack/browse/dist/browse` (~58MB, ~10 second build)

### 17.4 Connector Setup

```
Cowork → Customize → Plugins → gstack → Connectors
  1. Toggle the connector ON
  2. Authorize the OAuth integration
```

Available connectors (from `cowork/.mcp.json`):
```
github  → https://api.github.com/mcp
gitlab  → https://gitlab.com/api/mcp
linear  → https://mcp.linear.app/mcp
notion  → https://mcp.notion.com/mcp
slack   → https://mcp.slack.com/mcp
```

---

## 18. Quality Attributes

### 18.1 Reliability

- **Graceful degradation:** every skill works without any connector; no skill fails hard on a missing MCP server
- **Idempotency of writes:** Greptile history appends; QA reports are directory-scoped; no destructive overwrites
- **Test gate in /ship:** tests must pass before CHANGELOG/version/PR creation; failed tests hard-stop the workflow

### 18.2 Usability

- **Zero-configuration for conversational skills:** `/plan-*`, `/review`, `/ship`, `/retro` run with no setup
- **Single command, full output:** `/ship` runs end-to-end from one command to PR URL
- **Terse review output:** file:line + one-line fix, never verbose

### 18.3 Correctness

- **Full diff before commenting:** `/review` and `/ship` read the entire diff before flagging issues
- **Suppressions prevent repeated false positives:** Greptile history prevents re-raising dismissed issues
- **Bisectable commits:** `/ship` groups changes into logical, independently-valid commits

### 18.4 Extensibility

- Adding a new skill requires a new `cowork/skills/<name>/SKILL.md`
- Adding a new connector requires an entry in `cowork/.mcp.json` and documentation in `cowork/CONNECTORS.md`
- The Greptile history schema is forward-extensible (append-only, pipe-delimited, ignored unknown lines)

### 18.5 Performance

- Browser binary: ~3s cold start, ~100–200ms per command after warm
- `/ship` runs tests in parallel where possible (npm + rspec in parallel via `&` + `wait`)
- Greptile API calls run in parallel (line-level + top-level fetched simultaneously)

---

## 19. Constraints & Decisions

### 19.1 Key Design Decisions

| Decision | Rationale |
|----------|-----------|
| SKILL.md as executable specification | Claude Cowork's plugin model uses SKILL.md files as the instruction set. No separate runtime code needed for conversational skills. |
| Browser binary as compiled Bun/Playwright | Headless Chromium requires a compiled binary for performance and portability. The binary is pre-built and distributed via the gstack repo. |
| Greptile history as append-only flat file | Simple, human-readable, diff-friendly. No DB dependency. Resilient to partial writes. |
| MCP HTTP transport for all connectors | Claude Cowork's MCP integration is HTTP-based. All five connectors use standard HTTP MCP endpoints. |
| GitLab added alongside GitHub | CONNECTORS.md documents both source-control systems; `.mcp.json` now reflects both for parity. |
| Non-interactive /ship | "The user said /ship which means DO IT." Interrupts are expensive. Minimize to essential gates only. |
| Greptile auto-reply for already-fixed/FP in /ship | Ship is non-interactive; ALREADY-FIXED and FALSE-POSITIVE have deterministic responses. Only VALID issues need judgment. |

### 19.2 Constraints

| Constraint | Impact |
|------------|--------|
| Claude Cowork SKILL.md format | Skills must be written as natural-language instructions, not code |
| No runtime code for conversational skills | All logic expressed declaratively in SKILL.md |
| Browser binary: Bun v1.0+, macOS/Linux x64/arm64 | Windows not supported; requires Bun runtime |
| Greptile triage requires GitHub connection | Cannot triage GitLab MR comments (GitHub-only bot) |
| MCP servers optional | Skills must remain functional in standalone mode |

---

## 20. Glossary

| Term | Definition |
|------|------------|
| **Cowork** | Claude Cowork — Anthropic's collaborative Claude product with plugin support |
| **SKILL.md** | A Markdown file that serves as Claude's instruction set for a skill. Loaded by the Cowork runtime when a skill is invoked. |
| **Skill** | A named workflow or capability, implemented as a SKILL.md file, invocable via slash command or natural language |
| **MCP** | Model Context Protocol — the standard for connecting Claude to external tools and services |
| **Supercharged mode** | Skill execution with one or more MCP connectors active, enabling automatic data retrieval |
| **Standalone mode** | Skill execution without any MCP connectors; works from pasted or described input |
| **Greptile** | An AI code review bot (`greptile-apps[bot]`) that posts review comments on GitHub PRs |
| **Greptile triage** | The process of fetching, classifying (VALID/ALREADY-FIXED/FALSE-POSITIVE), and acting on Greptile bot comments |
| **Greptile history** | `~/.gstack/greptile-history.md` — a persistent log of Greptile comment triage outcomes, used to suppress known false positives |
| **ALREADY-FIXED** | A Greptile comment classification: the issue is real but was addressed in a later commit on the branch |
| **FALSE-POSITIVE** | A Greptile comment classification: the comment misunderstands the code or flags a non-issue |
| **VALID** | A Greptile comment classification: the issue is real and unaddressed in the current code |
| **SUPPRESSED** | A Greptile comment that matches a known false positive in the history file and is silently skipped |
| **Bisectable commit** | A commit that is independently valid (no broken imports, no dangling references) — enables `git bisect` to work correctly |
| **gstack browser binary** | The compiled Bun/Playwright binary at `~/.claude/skills/gstack/browse/dist/browse` that drives headless Chromium |
| **Diff-aware mode** | `/qa` mode that auto-detects the feature branch diff and tests only the pages changed by that diff |
| **Health score** | A 0–100 score produced by `/qa` across six quality dimensions (visual, functional, console, navigation, performance, content) |
| **VERSION file** | A file at the repository root containing the current version string (4-digit `MAJOR.MINOR.PATCH.MICRO` or semver) |
| **MICRO bump** | The smallest version increment (4th digit): < 50 lines changed, trivial tweaks |
| **PATCH bump** | Second-smallest increment: 50+ lines changed, bug fixes, small features |
| **@e ref** | A snapshot reference (e.g., `@e3`) returned by `$B snapshot -i`, uniquely identifying an interactive element for subsequent commands |
