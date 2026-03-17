# Connectors

This plugin works standalone (describe or paste your code/plan) and gets supercharged when you connect your tools.

## Supported Integrations

### Source Control

| Tool | What It Enables |
|------|----------------|
| **GitHub** | Pull PR diffs automatically, read commit history for `/retro`, post review comments, create PRs for `/ship` |
| **GitLab** | Pull MR diffs, read commit history |

Connect: [GitHub MCP](https://docs.github.com/en/developers/apps) | [GitLab MCP](https://docs.gitlab.com/)

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
