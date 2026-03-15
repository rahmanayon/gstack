# Connectors

This plugin works standalone (describe or paste your code/plan) and gets supercharged when you connect your tools.

## Supported Integrations

### Source Control

| Tool | What It Enables |
|------|----------------|
| **GitHub** | Pull PR diffs automatically, read commit history for `/retro`, post review comments, create PRs for `/ship`, read Greptile bot comments for `/review` and `/ship` |
| **GitLab** | Pull MR diffs, read commit history |

Connect: [GitHub MCP](https://docs.github.com/en/developers/apps) | [GitLab MCP](https://docs.gitlab.com/)

### Code Intelligence

| Tool | What It Enables |
|------|----------------|
| **Greptile** | AI-powered code review comments posted automatically on every PR; `/review` and `/ship` triage those comments — classifying each as VALID, ALREADY-FIXED, or FALSE-POSITIVE and suppressing known noise |

#### How Greptile Works with gstack

Greptile runs as a **GitHub App** (`greptile-apps[bot]`) that automatically reviews every pull request in your repository and posts inline comments. The gstack plugin uses your GitHub connection to read those comments and triage them — no separate Greptile auth needed for basic bot triage.

For advanced code intelligence (semantic search, cross-file analysis), connect the **Greptile MCP** server directly:
- MCP server URL: `https://api.greptile.com/mcp`
- Requires a Greptile API key (see setup steps below)

#### Step-by-step Greptile Setup

**Part 1 — Install the Greptile GitHub App (enables bot comments on PRs)**

1. Go to [app.greptile.com](https://app.greptile.com) and sign in with GitHub.
2. Click **Add Repository** and select the repos you want Greptile to review.
3. Greptile will index your codebase (~2–5 minutes for a typical repo).
4. From this point, every new PR in your repo will automatically receive Greptile review comments from `greptile-apps[bot]`.

> Greptile bot triage in `/review` and `/ship` requires only a GitHub connection — no Greptile API key needed for this step.

**Part 2 — Connect the Greptile MCP (enables direct code intelligence)**

1. Go to [app.greptile.com/settings/api](https://app.greptile.com/settings/api) and copy your API key.
2. In Cowork, open **Customize → Plugins → gstack → Connectors**.
3. Find **Greptile** and toggle it on.
4. Paste your Greptile API key when prompted.
5. Cowork connects to `https://api.greptile.com/mcp` using that key.

**Part 3 — Verify**

After setup, run `/review` on any open PR. You should see a `## Greptile Review` section at the end of the output listing any bot comments with their VALID / ALREADY-FIXED / FALSE-POSITIVE classification.

#### History & Suppression

When you mark a Greptile comment as a false positive (option C in the `/review` or `/ship` flow), gstack saves it to `~/.gstack/greptile-history.md`. Future runs automatically skip matching comments — you never see the same noise twice.

History file location: `~/.gstack/greptile-history.md`
Format: `<YYYY-MM-DD> | <owner/repo> | <type> | <file-pattern> | <category>`

### Project Tracker

| Tool | What It Enables |
|------|----------------|
| **Linear** | Link review findings to tickets, create follow-up issues from `/review` |
| **Jira / Atlassian** | Same as Linear |
| **Asana** | Same as Linear |

### Knowledge Base

| Tool | What It Enables |
|------|----------------|
| **Notion** | Read prior ADRs for `/plan-eng-review`, pull team coding standards for `/review`, architecture docs |
| **Confluence** | Same as Notion |

### Communication

| Tool | What It Enables |
|------|----------------|
| **Slack** | Post retro summaries to team channel, notify on QA report completion |

## How to Connect

1. Open Cowork → Customize → Plugins → gstack → Connectors
2. Toggle on the tools you want to connect
3. Authorize each integration

If a connector isn't connected, the skill will work in standalone mode — just describe your context or paste the relevant content.
