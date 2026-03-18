---
name: retro
description: Weekly engineering retrospective. Analyzes commit history, work patterns, and code quality metrics with persistent history and trend tracking. Team-aware — breaks down per-person contributions with praise and growth areas. Triggers on "retro", "retrospective", "weekly review", "team review", or "how did we do this week". Works with GitHub connected (automatic) or standalone (describe your week).
argument-hint: "[7d | 14d | 30d | 24h | compare | compare 14d]"
---

# /retro

> If you see unfamiliar placeholders or need to check which tools are connected, see [CONNECTORS.md](../../CONNECTORS.md).

Generates a comprehensive engineering retrospective analyzing commit history, work patterns, and code quality metrics. Team-aware: identifies each contributor and provides per-person praise and growth opportunities.

## Usage

```
/retro              ← last 7 days (default)
/retro 24h          ← last 24 hours
/retro 14d          ← last 14 days
/retro 30d          ← last 30 days
/retro compare      ← compare current period vs prior same-length period
/retro compare 14d  ← compare with explicit window
```

## How It Works

```
┌─────────────────────────────────────────────────────────────────┐
│                      /retro                                      │
├─────────────────────────────────────────────────────────────────┤
│  STANDALONE (always works)                                      │
│  ✓ Describe your week's work and I'll generate the retro       │
│  ✓ Paste commit logs if you have them                          │
│  ✓ Per-person praise and growth areas                          │
├─────────────────────────────────────────────────────────────────┤
│  SUPERCHARGED (when GitHub is connected)                        │
│  + Pull commit history automatically                            │
│  + Analyze PR patterns and review activity                      │
│  + Track PR review patterns and quality metrics over time       │
└─────────────────────────────────────────────────────────────────┘
```

## Step 1: Gather Raw Data

Parse the time window argument. Default: 7 days.

**If GitHub is connected:**
- List commits on the main branch for the time window
- Get commit stats (insertions, deletions) per author
- Get PR merges and review activity

**If working via git locally:**
```bash
git fetch origin main --quiet
git config user.name   # identify "you"
git config user.email

# All commits in window
git log origin/main --since="7 days ago" --format="%H|%aN|%ae|%ai|%s" --shortstat

# Per-commit LOC breakdown
git log origin/main --since="7 days ago" --format="COMMIT:%H|%aN" --numstat

# Per-author summary
git shortlog origin/main --since="7 days ago" -sn --no-merges
```

**If working standalone:**
Ask the user to describe:
- Who contributed and what they worked on
- Rough commit counts or PR counts if known
- Any notable wins or blockers

## Step 2: Compute Metrics

| Metric | Value |
|--------|-------|
| Commits to main | N |
| Contributors | N |
| PRs merged | N |
| Total insertions | N |
| Total deletions | N |
| Net LOC added | N |
| Active days | N |

**Per-author leaderboard:**
```
Contributor         Commits   +/-          Top area
You (name)               32   +2400/-300   app/services/
alice                    12   +800/-150    models/
bob                       3   +120/-40     tests/
```

Sort by commits descending. The current user (git config user.name or whoever ran the command) appears first, labeled "You (name)".

## Step 3: Commit Time Distribution

Show hourly histogram:
```
Hour  Commits  ████████████████
 07:    5      █████
 10:   12      ████████████
 14:    8      ████████
 21:    3      ███
```

Identify:
- Peak hours
- Dead zones
- Late-night coding clusters (after 10pm)
- Bimodal vs. continuous pattern

## Step 4: Work Session Detection

Detect sessions using 45-minute gap threshold between consecutive commits.

Classify sessions:
- **Deep sessions** (50+ min)
- **Medium sessions** (20-50 min)
- **Micro sessions** (<20 min)

Calculate:
- Total active coding time
- Average session length
- LOC per hour of active time

## Step 5: Commit Type Breakdown

Categorize by conventional commit prefix (feat/fix/refactor/test/chore/docs):
```
feat:     20  (40%)  ████████████████████
fix:      27  (54%)  ███████████████████████████
refactor:  2  ( 4%)  ██
```

Flag if fix ratio exceeds 50% — signals "ship fast, fix fast" pattern that may indicate review gaps.

## Step 6: Hotspot Analysis

Show top 10 most-changed files. Flag:
- Files changed 5+ times (churn hotspots)
- Test files vs production files in hotspot list

## Step 7: Focus Score + Ship of the Week

**Focus score:** Percentage of commits touching the single most-changed top-level directory. Higher = deeper focused work. Lower = scattered context-switching.

**Ship of the week:** Highest-LOC PR or commit cluster in the window. Highlight it with PR number/title (if available), LOC changed, and why it matters.

## Step 8: Per-Person Deep Dive

For each contributor (you + every teammate):

### [Name] — [N commits, +X/-Y LOC]

**Wins this week:**
- [Specific achievement with evidence, e.g., "Landed the payment redesign — 800 LOC, well-tested"]
- [Another win]

**Growth opportunity:**
- [One specific, actionable observation, e.g., "Fix:feat ratio was 3:1 this week — worth a retro on root causes"]

**Pattern:**
- [Work style observation, e.g., "Deep sessions, mostly afternoon — peak focus window identified"]

**Tone guidelines:**
- Be generous with praise — this is a celebratory document
- Growth opportunities should be specific and actionable, not vague criticism
- Focus on patterns and systemic improvements, not individual mistakes
- Frame everything as "how do we make the next week even better"

## Step 9: Compare Mode (if `compare` argument)

Run the same analysis for the prior same-length period. Then produce a comparison:

```
                     This period    Prior period    Delta
Commits                    45            38         +7 (+18%)
Contributors                3             3          =
PRs merged                 12             9         +3
Net LOC                 +2400          +1800        +600
Fix ratio                  40%           55%        -15% ✓
Focus score                62%           45%        +17% ✓
```

Call out:
- Improvements (✓)
- Regressions (⚠)
- Notable trends

## Final Output

End with:
```
## gstack Retro — [Date Range]

[All sections above]

---
**Overall:** [One sentence vibe check — e.g., "Strong week: high focus, good PR velocity, fix ratio improving."]
**Next week:** [1-2 concrete suggestions based on the data]
```

## If Connectors Available

If **GitHub** is connected:
- Pull commit history automatically (no manual git commands needed)
- Include PR titles, review counts, and merge times
- Show PR size distribution

If **Slack** is connected:
- Post the retro summary to your team channel

## Tips

1. **Run weekly.** The retro is most valuable when it's a habit.
2. **Compare periods.** Use `compare` to track trends over time.
3. **Share with the team.** Per-person praise is morale fuel.
4. **Pair with `/review`.** If fix ratio is high, run a deep review pass on the codebase.
