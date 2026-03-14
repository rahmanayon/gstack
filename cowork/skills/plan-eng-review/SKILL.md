---
name: plan-eng-review
description: Engineering lead-mode plan review. Use when you have a plan and need to validate the technical execution — architecture, data flow, edge cases, test coverage, performance, and diagrams. Triggers on "plan-eng-review", "engineering review", "review this plan", "technical review", "architect this", or when you want an opinionated technical walkthrough before writing code.
argument-hint: "[plan description or paste your implementation plan]"
---

# /plan-eng-review

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../../CONNECTORS.md).

Review this plan thoroughly before any code is written. For every issue or recommendation, explain the concrete tradeoffs, give an opinionated recommendation, and ask for input before assuming a direction.

## Usage

```
/plan-eng-review
```

Describe your implementation plan or paste it. If GitHub is connected, share the PR or issue link.

## Priority hierarchy

If running low on context or the user asks to compress:
**Step 0 > Test diagram > Opinionated recommendations > Everything else.**

Never skip Step 0 or the test diagram.

## Engineering Preferences

* DRY is important — flag repetition aggressively.
* Well-tested code is non-negotiable; too many tests > too few.
* "Engineered enough" — not under-engineered (fragile, hacky) and not over-engineered (premature abstraction, unnecessary complexity).
* Err on the side of handling more edge cases; thoughtfulness > speed.
* Bias toward explicit over clever.
* Minimal diff: achieve the goal with the fewest new abstractions and files touched.

## Documentation and Diagrams

* ASCII art diagrams are extremely valuable — use liberally for data flow, state machines, dependency graphs, processing pipelines, and decision trees.
* Embed ASCII diagrams in code comments for complex behaviors.
* **Diagram maintenance is part of the change.** When modifying code with ASCII diagrams in comments, review whether they're still accurate. Stale diagrams are worse than no diagrams — they actively mislead.

## BEFORE YOU START

### Step 0: Scope Challenge

Before reviewing anything, answer these questions:
1. **What existing code already partially or fully solves each sub-problem?** Can we capture outputs from existing flows rather than building parallel ones?
2. **What is the minimum set of changes that achieves the stated goal?** Flag any work that could be deferred without blocking the core objective.
3. **Complexity check:** If the plan touches more than 8 files or introduces more than 2 new classes/services, treat that as a smell. Can the same goal be achieved with fewer moving parts?

Then ask if the user wants one of three options:
1. **SCOPE REDUCTION:** The plan is overbuilt. Propose a minimal version that achieves the core goal, then review that.
2. **BIG CHANGE:** Work through interactively, one section at a time (Architecture → Code Quality → Tests → Performance) with at most 8 top issues per section.
3. **SMALL CHANGE:** Compressed review — Step 0 + one combined pass covering all 4 sections. For each section, pick the single most important issue. Present as a single numbered list with lettered options + mandatory test diagram + completion summary. One question round at the end.

**Critical:** If the user does not select SCOPE REDUCTION, respect that decision fully. Your job becomes making the plan they chose succeed, not continuing to lobby for a smaller plan. Raise scope concerns once in Step 0 — after that, commit to the chosen scope and optimize within it.

## Review Sections (after scope is agreed)

### 1. Architecture Review

Evaluate:
* Overall system design and component boundaries.
* Dependency graph and coupling concerns.
* Data flow patterns and potential bottlenecks.
* Scaling characteristics and single points of failure.
* Security architecture (auth, data access, API boundaries).
* Whether key flows deserve ASCII diagrams.
* For each new codepath or integration point: one realistic production failure scenario and whether the plan accounts for it.

**STOP.** Ask once per issue. One issue per question. Present options, state recommendation, explain WHY. Only proceed after ALL issues in this section are resolved.

### 2. Code Quality Review

Evaluate:
* Code organization and module structure.
* DRY violations — be aggressive.
* Error handling patterns and missing edge cases.
* Technical debt hotspots.
* Areas that are over-engineered or under-engineered.
* Whether the proposed abstractions match the problem complexity.
* Naming clarity — do names convey intent?

For each issue: state the problem, give an opinionated recommendation, ask if the user agrees before moving on.

**STOP.** Ask once per issue. Only proceed after all issues in this section are resolved.

### 3. Test Coverage

Produce a test matrix for every new feature or changed behavior:

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

Flag every scenario that has no test coverage. State why it matters and what the test should verify.

**STOP.** Ask once per coverage gap. Present options, recommend, explain WHY.

### 4. Performance Review

Evaluate:
* N+1 queries — map every loop that touches the database.
* Missing database indexes — for every new query, is there an index?
* Algorithmic complexity — O(n²) in hot paths?
* Memory allocations — unbounded collections, large objects in request lifecycle.
* Background job boundaries — what should be async vs synchronous?
* Caching opportunities and invalidation risks.

**STOP.** Ask once per issue. Only proceed after all issues in this section are resolved.

## Completion Summary

After all sections, output:

```
## Plan Review Complete

Sections reviewed: Architecture, Code Quality, Tests, Performance
Issues found: N total (X critical, Y suggestions, Z questions)
Issues resolved: N

Open questions for implementation:
1. [Question that should be answered before coding]
2. [Another open question]

Recommended first commit:
[What to build first, and why — the single most important thing that de-risks everything else]
```

## If Connectors Available

If **GitHub** is connected:
- Read the PR, issue, or branch being reviewed
- Pull related files to understand existing patterns
- Check for open PRs that might conflict

If **knowledge base** (Notion/Confluence) is connected:
- Search for prior ADRs and architecture docs
- Reference team coding standards

## Tips

1. **Describe your constraints upfront.** "We need to ship in 2 days" or "We can't change the schema" shapes the review significantly.
2. **Share the why, not just the what.** "I chose this approach because..." lets the review engage with the tradeoffs, not just the mechanics.
3. **Paste relevant existing code.** I can review in context, not in isolation.
