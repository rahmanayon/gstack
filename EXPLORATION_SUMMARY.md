# gstack Repository Comprehensive Overview

## 1. Directory Structure (2-3 Levels Deep)

```
/home/runner/work/gstack/gstack/
├── browse/                          # Headless browser CLI (Playwright-based)
│   ├── src/                         # TypeScript source
│   │   ├── commands.ts              # Command registry (single source of truth)
│   │   ├── snapshot.ts              # Snapshot system + SNAPSHOT_FLAGS metadata
│   │   ├── server.ts                # Bun HTTP server (localhost daemon)
│   │   ├── browser-manager.ts       # Playwright browser lifecycle
│   │   ├── cli.ts                   # CLI entry point
│   │   ├── find-browse.ts           # Binary locator
│   │   ├── read/write/meta-commands.ts  # Command implementations
│   │   ├── cookie-import-browser.ts # Browser cookie import UI
│   │   └── config.ts                # Configuration
│   ├── bin/                         # Scripts
│   ├── dist/                        # Compiled binary (gitignored)
│   ├── test/                        # Integration tests
│   │   ├── fixtures/
│   │   ├── find-browse.test.ts
│   │   ├── snapshot.test.ts
│   │   └── config.test.ts
│   ├── SKILL.md                     # Generated (do not edit)
│   └── SKILL.md.tmpl                # Template source
│
├── ship/                            # Ship workflow skill
│   └── SKILL.md                     # Fully automated ship workflow
│
├── review/                          # PR review skill
│   ├── SKILL.md
│   ├── checklist.md
│   └── greptile-triage.md
│
├── plan-ceo-review/                 # CEO/founder mode plan review
│   └── SKILL.md
│
├── plan-eng-review/                 # Engineering manager mode plan review
│   └── SKILL.md
│
├── retro/                           # Engineering retrospective skill
│   └── SKILL.md
│
├── qa/                              # Systematic QA testing skill
│   ├── SKILL.md
│   ├── SKILL.md.tmpl
│   ├── references/
│   │   └── issue-taxonomy.md
│   └── templates/
│       └── qa-report-template.md
│
├── setup-browser-cookies/           # Cookie import session manager
│   └── SKILL.md
│
├── gstack-upgrade/                  # Upgrade utility
│   └── SKILL.md
│
├── scripts/                         # Build & DX tools
│   ├── gen-skill-docs.ts            # Template → SKILL.md generator
│   ├── skill-check.ts               # Health dashboard
│   ├── skill-parser.ts              # SKILL.md validator
│   └── dev-skill.ts                 # Watch mode
│
├── test/                            # Skill validation & tests
│   ├── helpers/
│   │   ├── skill-parser.ts
│   │   └── session-runner.ts
│   ├── skill-validation.test.ts     # Static command validation
│   ├── gen-skill-docs.test.ts       # Generator + quality evals
│   ├── skill-e2e.test.ts            # Agent SDK E2E tests
│   └── skill-llm-eval.test.ts       # LLM-as-judge evals
│
├── bin/                             # Entry points
│   ├── dev-setup
│   └── gstack-update-check
│
├── setup                            # Setup script (build binary + symlink skills)
├── setup-browser-cookies            # Browser cookie import wrapper
├── gstack-upgrade                   # Upgrade runner
├── conductor.json                   # Conductor config for parallel sessions
├── package.json                     # Build scripts & dependencies
├── SKILL.md                         # Browse skill (auto-generated)
├── SKILL.md.tmpl                    # Browse skill template
├── ARCHITECTURE.md                  # Design & architecture docs
├── BROWSER.md                       # Browser usage guide
├── CLAUDE.md                        # Development mode instructions
├── README.md                        # Installation & overview
├── CONTRIBUTING.md                  # Contribution guidelines
├── CHANGELOG.md                     # Version history
├── TODO.md / TODOS.md              # Development tasks
└── VERSION                          # Version number
```

---

## 2. Major Directories & What They Do

### **browse/**
**Purpose:** Fast headless Chromium browser for QA testing and site dogfooding.
- **Key features:**
  - ~100-200ms per command after first call (~3s startup)
  - Persistent state (cookies, tabs, sessions) between calls
  - 30-min auto-shutdown after idle
  - Compiled native binary (~58MB)
  - Ref-based element selection (@e1, @e2 refs)
  - Snapshot system with interactive-only, compact, diff, annotate modes
  - Daemon model (HTTP server on localhost with Bearer token auth)
  
**Tech:** Playwright (Chromium), TypeScript, Bun compiled binary

### **ship/**
**Purpose:** Fully automated ship workflow for ready branches.
- **What it does:**
  - Merge main and resolve conflicts
  - Run tests
  - Review diff (via Greptile)
  - Auto-bump VERSION file (MICRO/PATCH/MINOR/MAJOR)
  - Update CHANGELOG from git diff
  - Create bisectable commits
  - Push and open PR
- **Decision:** Non-interactive (user said /ship = do it)
- **Stops only for:** Merge conflicts, test failures, CRITICAL review issues

### **review/**
**Purpose:** Paranoid staff engineer code review.
- **What it does:**
  - Deep bug hunting (not just style)
  - Catches race conditions, trust boundaries, missing cleanup
  - Triages Greptile review comments
  - Produces structured feedback
- **Files:** SKILL.md, checklist.md, greptile-triage.md

### **plan-ceo-review/**
**Purpose:** CEO/founder-mode plan review - rethink the problem.
- **Philosophy:** Find the 10-star product hiding in the request
- **Three modes:**
  - SCOPE EXPANSION: Dream big, envision the platonic ideal
  - HOLD SCOPE: Lock in scope, catch every failure mode
  - SCOPE REDUCTION: Strip to essentials, be ruthless
- **No code changes** — planning only

### **plan-eng-review/**
**Purpose:** Engineering manager/tech lead plan review - lock in architecture.
- **What it covers:**
  - Architecture & data flow diagrams
  - Edge cases & failure modes
  - Test matrix & observability
  - Error handling per specific exception class
  - Data flow shadow paths (nil, empty, upstream error)

### **retro/**
**Purpose:** Weekly engineering retrospective with trend tracking.
- **What it does:**
  - Analyzes commit history & work patterns
  - Code quality metrics per contributor
  - Persistent JSON snapshots at `.context/retros/` for trends
  - Per-person praise and growth opportunities
  - Supports time windows: 24h, 7d (default), 14d, 30d, compare modes
  - Reports in Pacific time

### **qa/**
**Purpose:** Systematic QA testing with multiple modes.
- **Modes:**
  - **diff-aware** (auto on feature branches): Analyzes git diff, identifies affected pages
  - **full:** Systematic exploration of entire app
  - **quick:** 30-second smoke test
  - **regression:** Compare against stored baseline
- **Outputs:** Health score, screenshots, structured report with repro steps
- **Files:** SKILL.md.tmpl (generated), references/issue-taxonomy.md, templates/qa-report-template.md

### **setup-browser-cookies/**
**Purpose:** Session manager - import cookies from real browsers.
- **Supports:** Comet, Chrome, Arc, Brave, Edge
- **Workflow:** Cookie picker UI → import into headless session → test authenticated pages

---

## 3. Key Files Content

### **SKILL.md.tmpl** (Root-level)
**Purpose:** Template for browse skill documentation (generates SKILL.md).
Contains placeholders:
- `{{UPDATE_CHECK}}` → Substituted with update check code
- `{{BROWSE_SETUP}}` → Substituted with setup instructions
- `{{SNAPSHOT_FLAGS}}` → Substituted with snapshot flag table
- `{{COMMAND_REFERENCE}}` → Substituted with command reference table

**Structure:**
1. YAML frontmatter: name, version, description, allowed-tools
2. Update check section
3. Setup section (find binary, build if needed)
4. QA Workflows (test flow, deploy verification, dogfooding, responsive, upload, forms, dialogs, auth, compare, chain)
5. Assertion patterns
6. Snapshot system docs
7. Command reference (organized by category)
8. Tips

### **SKILL.md** (Root-level, Generated)
Same as template but with placeholders substituted in. **Do not edit directly.**

Regenerate with: `bun run gen:skill-docs`

### **package.json**
```json
{
  "name": "gstack",
  "version": "0.3.3",
  "scripts": {
    "build": "bun run gen:skill-docs && bun build --compile browse/src/cli.ts --outfile browse/dist/browse && ...",
    "gen:skill-docs": "bun run scripts/gen-skill-docs.ts",
    "dev": "bun run browse/src/cli.ts",
    "server": "bun run browse/src/server.ts",
    "test": "bun test browse/test/ test/ --ignore test/skill-e2e.test.ts --ignore test/skill-llm-eval.test.ts",
    "test:e2e": "SKILL_E2E=1 bun test test/skill-e2e.test.ts",
    "test:eval": "bun test test/skill-llm-eval.test.ts",
    "test:all": "bun test browse/test/ test/ ...",
    "skill:check": "bun run scripts/skill-check.ts",
    "dev:skill": "bun run scripts/dev-skill.ts",
    "start": "bun run browse/src/server.ts"
  },
  "dependencies": {
    "playwright": "^1.58.2",
    "diff": "^7.0.0"
  },
  "devDependencies": {
    "@anthropic-ai/claude-agent-sdk": "^0.2.75",
    "@anthropic-ai/sdk": "^0.78.0"
  }
}
```

### **browse/src/commands.ts** (First 100 lines)
Defines command registry as single source of truth:
- `READ_COMMANDS`: text, html, links, forms, accessibility, js, eval, css, attrs, console, network, cookies, storage, perf, dialog, is
- `WRITE_COMMANDS`: goto, back, forward, reload, click, fill, select, hover, type, press, scroll, wait, viewport, cookie, cookie-import, cookie-import-browser, header, useragent, upload, dialog-accept, dialog-dismiss
- `META_COMMANDS`: tabs, tab, newtab, closetab, status, stop, restart, screenshot, pdf, responsive, chain, diff, url, snapshot
- `COMMAND_DESCRIPTIONS`: Maps each command to category, description, and usage

**Architecture note:** Used by:
- `server.ts` (runtime dispatch)
- `gen-skill-docs.ts` (doc generation)
- `skill-parser.ts` (validation)
- `skill-check.ts` (health reporting)

### **browse/src/snapshot.ts** (First 100 lines)
Implements snapshot accessibility tree with ref-based element selection.
- **Architecture:** No DOM mutation
  1. `page.locator(scope).ariaSnapshot()` → YAML-like accessibility tree
  2. Parse tree, assign refs @e1, @e2, ...
  3. Build Playwright Locator for each ref
  4. Store Map<string, Locator> on BrowserManager
  5. Return compact text output with refs prepended

- **SNAPSHOT_FLAGS:** Array of 8 flags (metadata for CLI parsing & doc generation)
  - `-i/--interactive`: Interactive elements only (buttons, links, inputs)
  - `-c/--compact`: No empty structural nodes
  - `-d/--depth <N>`: Limit tree depth
  - `-s/--selector <sel>`: Scope to CSS selector
  - `-D/--diff`: Unified diff against last snapshot
  - `-a/--annotate`: Annotated screenshot with overlay boxes
  - `-o/--output <path>`: Output path for annotated screenshot
  - `-C/--cursor-interactive`: Cursor-interactive elements (@c refs)

---

## 4. SKILL.md Pattern & Format

### Structure Template
All SKILL.md files follow this pattern:

```markdown
---
name: <skill-name>
version: <x.y.z>
description: |
  Multi-line description
  explaining what this skill does
allowed-tools:
  - Bash
  - Read
  - [other tools]
---

## Update Check (run first)

[Inline bash code to check for upgrades]

# <Skill Title>

[Main skill instructions and workflows]

## Step 1: [Setup]
...

## Step 2: [Main Logic]
...

## Tips
- Key patterns and advice
```

### Key patterns:
1. **YAML frontmatter** at top with metadata
2. **Update check** section runs first (checks ~/.claude/skills/gstack/bin/gstack-update-check)
3. **Setup** section with conditional logic (e.g., "if NEEDS_SETUP")
4. **Main workflow** broken into steps
5. **Inline code blocks** with bash/pseudocode
6. **No edit warnings** on generated files (SKILL.md)

### Generated vs Template files:
- `.tmpl` files have placeholder markers: `{{MARKER}}`
- Generated `.md` files have these substituted
- Regenerate: `bun run gen:skill-docs`
- **Commit both** (template source + generated output)

---

## 5. Cowork/Claude Related Files

Found:
- **CLAUDE.md** (root level) - Development mode instructions
  - Commands for local development
  - Project structure explanation
  - Watch mode, browser interaction notes
  - Deploying to active skill

- **README.md** - Installation and overview mentions "Claude Code" repeatedly
  - References Claude Code system: /skills installation
  - Eight opinionated workflow skills
  - Instructions include: "add 'gstack' section to CLAUDE.md that says to use the /browse skill from gstack"

- **ARCHITECTURE.md** - Technical documentation
  - Explains daemon model and why
  - Security model details
  - Cookie handling

**No "cowork" references found** - but all skills designed to work with Claude Code agent system

---

## 6. Skill Registration & Setup

### Setup File (`/home/runner/work/gstack/gstack/setup`)
Four-stage setup script:

**Stage 1: Build browse binary (smart rebuild)**
- Checks if browse/dist/browse exists and is executable
- Rebuilds if:
  - Binary missing OR
  - browse/src files newer than binary OR
  - package.json newer than binary OR
  - bun.lock newer than binary
- Runs: `bun install` && `bun run build`
- Writes `.version` file with git commit SHA

**Stage 2: Ensure Playwright Chromium**
- Checks if Chromium can launch
- If fails: `bunx playwright install chromium`

**Stage 3: Create skill symlinks** (only if in .claude/skills directory)
- Scans gstack dir for subdirs with SKILL.md
- Creates symlinks: `.claude/skills/<skill-name>` → `gstack/<skill-name>`
- Example: `ln -snf gstack/browse ~/.claude/skills/browse`

**Stage 4: Welcome & legacy cleanup**
- Creates ~/.gstack directory for state files
- Removes stale version check cache

### State File: `~/.gstack/browse.json`
```json
{
  "pid": 12345,
  "port": 34567,
  "token": "uuid-v4",
  "startedAt": "2024-01-01T12:00:00Z",
  "binaryVersion": "abc123def456"
}
```
- Atomic write (tmp + rename, mode 0o600)
- Allows version auto-restart detection

### How Skills Get Registered
1. Symlinks created at `~/.claude/skills/<skill-name>` → `gstack/<skill-name>`
2. Each skill dir has SKILL.md (Markdown prompt file)
3. Claude Code reads from `~/.claude/skills/` directory
4. Each `.md` file becomes a `/slash-command`
5. Frontmatter metadata tells Claude what tools it can use

### Skill List (8 total)
1. `/browse` - browse/SKILL.md
2. `/ship` - ship/SKILL.md
3. `/review` - review/SKILL.md
4. `/plan-ceo-review` - plan-ceo-review/SKILL.md
5. `/plan-eng-review` - plan-eng-review/SKILL.md
6. `/qa` - qa/SKILL.md
7. `/retro` - retro/SKILL.md
8. `/setup-browser-cookies` - setup-browser-cookies/SKILL.md
9. Bonus: `/gstack-upgrade` - gstack-upgrade/SKILL.md

---

## 7. Testing Patterns

### Test Organization
- **browse/test/** - Integration tests for browse commands
- **test/** - Skill validation and E2E tests

### Three Tiers of Testing

**Tier 1: Static Validation**
- File: `test/skill-validation.test.ts`
- What: Validates all `$B` commands in SKILL.md are registered
- What: Validates all snapshot flags are real
- What: Command registry consistency checks
- Coverage: ~10 test cases

**Tier 2: Generator & Quality**
- File: `test/gen-skill-docs.test.ts`
- What: Tests that SKILL.md.tmpl → SKILL.md generation works
- What: Validates placeholders are substituted
- What: Checks doc quality

**Tier 3: E2E (optional)**
- File: `test/skill-e2e.test.ts`
- What: Spawns real Claude Code session, runs `/qa` command
- Cost: ~$0.50 per run
- Time: ~60s
- Enabled: `SKILL_E2E=1 bun test test/skill-e2e.test.ts`

**Tier 4: LLM-as-Judge (optional)**
- File: `test/skill-llm-eval.test.ts`
- What: LLM judges quality of skill outputs
- Enabled: `bun run test:eval`
- Requires: ANTHROPIC_API_KEY

### Example Test (find-browse.test.ts)
```typescript
import { describe, test, expect } from 'bun:test';
import { locateBinary } from '../src/find-browse';
import { existsSync } from 'fs';

describe('locateBinary', () => {
  test('returns null when no binary exists at known paths', () => {
    const result = locateBinary();
    expect(result === null || typeof result === 'string').toBe(true);
  });

  test('returns string path when binary exists', () => {
    const result = locateBinary();
    if (result !== null) {
      expect(existsSync(result)).toBe(true);
    }
  });
});
```

### Example Test (skill-validation.test.ts)
```typescript
test('all $B commands in SKILL.md are valid browse commands', () => {
  const result = validateSkill(path.join(ROOT, 'SKILL.md'));
  expect(result.invalid).toHaveLength(0);
  expect(result.valid.length).toBeGreaterThan(0);
});

test('COMMAND_DESCRIPTIONS covers all commands in sets', () => {
  const allCmds = new Set([...READ_COMMANDS, ...WRITE_COMMANDS, ...META_COMMANDS]);
  const descKeys = new Set(Object.keys(COMMAND_DESCRIPTIONS));
  for (const cmd of allCmds) {
    expect(descKeys.has(cmd)).toBe(true);
  }
});
```

### Test Helpers
- `skill-parser.ts`: Parses SKILL.md, extracts `$B` commands, validates them
- `session-runner.ts`: Spawns Claude Code sessions for E2E tests

### Running Tests
```bash
bun test                       # Tiers 1-2 (fast, ~30s)
bun run test:e2e              # Tier 3 (slow, ~$0.50, requires SKILL_E2E=1)
bun run test:eval             # Tier 4 (requires ANTHROPIC_API_KEY)
bun run test:all              # All tiers
```

---

## 8. Build & Doc Generation

### Build Process
```bash
bun run build
```
This does:
1. `gen:skill-docs` - Regenerate all SKILL.md from .tmpl templates
2. `bun build --compile browse/src/cli.ts --outfile browse/dist/browse` - Compile CLI binary
3. `bun build --compile browse/src/find-browse.ts --outfile browse/dist/find-browse` - Compile helper
4. `git rev-parse HEAD > browse/dist/.version` - Write version
5. Cleanup `.*.bun-build` files

### Doc Generation (gen-skill-docs.ts)
Reads .tmpl files, substitutes placeholders:
- `{{UPDATE_CHECK}}` → Update check bash code
- `{{BROWSE_SETUP}}` → Setup detection code
- `{{SNAPSHOT_FLAGS}}` → Table of snapshot flags
- `{{COMMAND_REFERENCE}}` → Organized command table

**Generates:**
- browse/SKILL.md (from browse/SKILL.md.tmpl)
- qa/SKILL.md (from qa/SKILL.md.tmpl)
- Root SKILL.md (from SKILL.md.tmpl)

---

## 9. Architecture & Design Decisions

### Why Daemon Model?
- **Sub-second latency:** After startup, ~100-200ms per command
- **Persistent state:** Cookies, tabs, localStorage, login sessions persist
- **Efficient QA:** 20+ commands without restart overhead

### Why Bun?
1. **Compiled binaries:** Single ~58MB executable, no node_modules at runtime
2. **Native SQLite:** Cookie decryption reads Chromium's SQLite directly
3. **Native TypeScript:** Source files run directly, no compilation step
4. **Fast HTTP server:** Bun.serve() built-in, no Express/Fastify needed

### Why Playwright?
- Cross-platform headless browser support
- Persistent state management
- Accessibility tree (ariaSnapshot)
- Screenshot + PDF generation

### Security Model
- **Localhost only:** Not reachable from network
- **Bearer token auth:** Random UUID per session, mode 0o600 file
- **Keychain access requires user approval** on first cookie import
- **Cookie values never written to disk** in plaintext (decrypted in-process)

### Port Selection
- Random port 10000-60000 (retry 5 times on collision)
- Allows 10+ parallel Conductor workspaces with zero conflicts

### Version Auto-Restart
- Stores `binaryVersion` in state file
- On CLI invocation, if versions differ, kills old server, starts new one
- Prevents "stale binary" bugs

---

## 10. Directory Sizes

| Directory | File Count | Purpose |
|-----------|-----------|---------|
| browse/src/ | 14 files | CLI + server implementation |
| browse/test/ | 8 files | Integration tests |
| test/ | 6 files | Skill validation + E2E |
| qa/ | 2 files + 2 dirs | QA skill + templates |
| plan-ceo-review/ | 1 file | Plan review skill |
| plan-eng-review/ | 1 file | Engineering review skill |
| retro/ | 1 file | Retro skill |
| ship/ | 1 file | Ship workflow skill |
| review/ | 3 files | PR review skill |
| setup-browser-cookies/ | 1 file | Cookie import skill |

---

## Summary

**gstack** is a comprehensive AI engineering workflow toolkit for Claude Code:

1. **Core:** Persistent headless Chromium daemon (browse/) with sub-second latency
2. **Workflow Skills:** 8 slash-commands for planning, reviewing, shipping, QA, and retrospectives
3. **Architecture:** Template-driven SKILL.md generation, compiled native binary, stateful daemon
4. **Testing:** 4-tier validation (static → E2E → LLM evals)
5. **Setup:** Single-file `./setup` script handles build, symlinks, and registration
6. **No "cowork" but:** All skills designed for Claude Code integration via skill symlinks

**Key insight:** Everything is about removing context switches. Each skill is a cognitive mode (founder, engineer, reviewer, QA lead). Claude picks the right tool for the right job.

