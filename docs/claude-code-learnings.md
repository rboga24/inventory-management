# Claude Code — Learnings Summary

> Hands-on training completed 2026-06-24 on the Factory Inventory Management System (Vue 3 + FastAPI). This document captures what was learned, how each feature works, and how to apply it to future projects.

---

## Table of Contents

1. [Project Setup & CLAUDE.md](#1-project-setup--claudemd)
2. [Skills — Task-Specific Instructions](#2-skills--task-specific-instructions)
3. [Subagents — Specialized Worker Agents](#3-subagents--specialized-worker-agents)
4. [Hooks — Automated Event Responses](#4-hooks--automated-event-responses)
5. [MCP Servers — External Tool Integration](#5-mcp-servers--external-tool-integration)
6. [GitHub Integration — @claude on PRs](#6-github-integration--claude-on-prs)
7. [Plugins — Community Workflow Packages](#7-plugins--community-workflow-packages)
8. [Agent Teams — Coordinated Multi-Agent Workflows](#8-agent-teams--coordinated-multi-agent-workflows)
9. [Worktrees — Isolated Parallel Branches](#9-worktrees--isolated-parallel-branches)
10. [Memory — Persistent Context Across Sessions](#10-memory--persistent-context-across-sessions)
11. [Quick Reference — File Locations](#11-quick-reference--file-locations)
12. [Applying to a New Project — Starter Checklist](#12-applying-to-a-new-project--starter-checklist)

---

## 1. Project Setup & CLAUDE.md

### What it is

`CLAUDE.md` is the project-level instruction file that Claude reads at the start of every session. It defines the project's stack, conventions, rules, and which tools/agents to use.

### What we learned

- Place `CLAUDE.md` at the repo root for project-wide instructions
- Place `client/CLAUDE.md` for frontend-specific guidance (subdirectory scoping)
- Define mandatory rules (e.g., "ANY .vue file modification MUST use vue-expert agent")
- List API endpoints, common issues, and file locations for quick reference
- Keep it updated as the project evolves — Claude reads it every session

### How to apply to future projects

```markdown
# CLAUDE.md

## Stack

- Frontend: [framework] (port X)
- Backend: [framework] (port Y)

## Quick Start

[How to run the project]

## Key Patterns

[Data flow, state management, API conventions]

## Common Issues

[Known gotchas, validation rules, edge cases]

## Subagents

[Which agents to use for what tasks]
```

### Key insight

CLAUDE.md is the single most impactful file you can create. A well-written one saves you from re-explaining context every session. Treat it like onboarding docs for a new developer.

---

## 2. Skills — Task-Specific Instructions

### What it is

Skills are `.claude/skills/<name>/SKILL.md` files containing detailed instructions Claude loads on demand. Unlike CLAUDE.md (always loaded), skills activate only when relevant.

### What we built

- **`vue-analyzer`** — Reads all Vue components, checks for performance issues (missing computed properties, v-for with index keys, heavy template logic), identifies code reuse opportunities (shared composables, base components), and generates a structured report
- **`backend-api-test`** — Guidelines for writing pytest tests with FastAPI TestClient

### Structure

```
.claude/skills/
├── vue-analyzer/
│   └── SKILL.md          ← frontmatter (name, description) + instructions
└── backend-api-test/
    └── SKILL.md
```

### SKILL.md format

```markdown
---
name: skill-name
description: One-line description — Claude uses this to decide when to load the skill
---

# Skill Title

[Detailed instructions, patterns, examples, output format]
```

### How to apply to future projects

Create skills for **repeatable analysis or generation tasks** specific to your project:

- **Database migration reviewer** — checks for missing indexes, breaking changes, rollback plan
- **API design validator** — verifies RESTful conventions, pagination, error formats
- **Component generator** — creates components following your project's exact patterns
- **Test writer** — generates tests matching your project's testing framework and conventions

### Key insight

The `description` field in frontmatter is critical — Claude uses it to decide whether to load the skill. Make it specific: "Analyze Vue 3 components for performance" is better than "Code analysis."

---

## 3. Subagents — Specialized Worker Agents

### What it is

Agent definition files (`.claude/agents/<name>.md`) that create specialized Claude instances with specific tools, models, and expertise. The parent Claude delegates tasks to them and gets results back.

### What we built

- **`debugger`** — Investigates runtime errors with Read, Grep, Glob, Bash. Has error classification tables for Vue and FastAPI, debugging commands, and a structured bug report output format
- **`code-reviewer`** — Reviews code quality with Read, Grep, Glob. Checks correctness, framework patterns, performance, and project-specific conventions
- **`vue-expert`** — Creates/modifies .vue files with full tool access including Playwright for browser testing

### Agent file format

```markdown
---
name: agent-name
description: What this agent specializes in
tools: Read, Grep, Glob, Bash # which tools it can use
model: sonnet # which model (sonnet, opus, haiku)
color: red # UI color in agent panel
---

# Agent Title

[System prompt: expertise, investigation process, output format,
project-specific knowledge, common patterns]
```

### How agents differ from skills

| Aspect        | Skill                        | Agent                               |
| ------------- | ---------------------------- | ----------------------------------- |
| What it is    | Instructions Claude follows  | A separate Claude instance          |
| Has tools     | No — uses parent's tools     | Yes — its own tool set              |
| Has model     | No — uses parent's model     | Yes — can use a different model     |
| Communication | Loaded into parent's context | Runs independently, returns results |
| Best for      | "How to do X" instructions   | "Do X for me" delegation            |

### How to apply to future projects

Create agents for roles that benefit from **isolation and specialization**:

- **`migration-checker`** — Validates database migrations before applying (Read, Grep, Bash)
- **`api-tester`** — Runs API endpoint tests and reports results (Read, Bash)
- **`dependency-auditor`** — Checks for outdated/vulnerable packages (Read, Grep, Bash)
- **`log-analyzer`** — Reads server logs and diagnoses issues (Read, Grep, Bash)

### Key insight

Include **project-specific knowledge** in the agent file — known gotchas, architecture diagrams, data flow patterns. A debugger agent that knows "SKU mismatches between demand_forecasts.json and inventory.json are a common issue" is far more useful than a generic one.

---

## 4. Hooks — Automated Event Responses

### What it is

Shell commands that run automatically in response to Claude Code events (file edits, tool calls, session start, etc.). Configured in `.claude/settings.json`.

### What we built

- **Prettier auto-format** — Runs `npx prettier --write` on every file after Edit or Write tool use

### Configuration

```json
// .claude/settings.json (project-scoped)
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [
          {
            "type": "command",
            "command": "npx prettier --write \"$CLAUDE_FILE_PATH\" 2>/dev/null || true"
          }
        ]
      }
    ]
  }
}
```

### Available hook events

| Event              | When it fires             | Common use                              |
| ------------------ | ------------------------- | --------------------------------------- |
| `PreToolUse`       | Before a tool runs        | Block dangerous commands, log activity  |
| `PostToolUse`      | After a tool succeeds     | Format code, run linters, trigger tests |
| `Stop`             | When Claude stops         | Show summary, save session notes        |
| `PreCompact`       | Before context compaction | Preserve key information                |
| `SessionStart`     | When session begins       | Set up environment, show reminders      |
| `UserPromptSubmit` | When user sends a message | Input validation, logging               |

### How to apply to future projects

- **Auto-format on save**: `PostToolUse` + `Edit|Write` → run your formatter (prettier, black, gofmt)
- **Auto-lint**: `PostToolUse` + `Edit|Write` → run eslint/pylint and feed errors back
- **Auto-test**: `PostToolUse` + `Edit|Write` → run relevant test file when source changes
- **Command logging**: `PreToolUse` + `Bash` → log all shell commands for audit
- **Dangerous command blocker**: `PreToolUse` + `Bash` → block `rm -rf`, `DROP TABLE`, etc.

### Key insight

The `2>/dev/null || true` suffix is important — without it, a hook failure blocks Claude from continuing. Use it for non-critical hooks (formatting). Remove it for hooks that should block on failure (security checks).

---

## 5. MCP Servers — External Tool Integration

### What it is

Model Context Protocol servers that give Claude access to external tools — browsers, GitHub, databases, etc. Configured in `.mcp.json` at the repo root.

### What we configured

- **Playwright** — Browser automation for testing UI (navigate, click, screenshot, fill forms)
- **GitHub** — PR creation, issue management, code search via GitHub API

### Configuration

```json
// .mcp.json (repo root)
{
  "mcpServers": {
    "playwright": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@playwright/mcp@latest"]
    },
    "github": {
      "type": "stdio",
      "command": "npx",
      "args": ["-y", "@modelcontextprotocol/server-github"],
      "env": {
        "GITHUB_PERSONAL_ACCESS_TOKEN": "${GITHUB_PERSONAL_ACCESS_TOKEN}"
      }
    }
  }
}
```

### How we used Playwright

```
1. Navigate to pages → verify they load
2. Take screenshots → visual verification
3. Click buttons → test interactions (Place Order, Export CSV)
4. Read page snapshots → verify data content
```

### How to apply to future projects

| MCP Server          | Use case                         | Install                                   |
| ------------------- | -------------------------------- | ----------------------------------------- |
| **Playwright**      | Browser testing, UI verification | `npx @playwright/mcp@latest`              |
| **GitHub**          | PR management, issue tracking    | `@modelcontextprotocol/server-github`     |
| **Postgres/SQLite** | Database queries                 | `@modelcontextprotocol/server-postgres`   |
| **Filesystem**      | File operations outside repo     | `@modelcontextprotocol/server-filesystem` |

### Key insight

MCP servers load at session startup. If you add one mid-session, you need to restart Claude Code. Also, the npx cache can get corrupted — if an MCP server fails to start, try `rm -rf ~/.npm/_npx/<hash>` and retry.

---

## 6. GitHub Integration — @claude on PRs

### What it is

GitHub Actions workflows that let you tag `@claude` in PR comments to get AI code review, and auto-review new PRs on open.

### What we set up

Two workflow files on the `main` branch:

**`.github/workflows/claude.yml`** — Responds to `@claude` mentions:

```yaml
name: Claude Code
on:
  issue_comment:
    types: [created]
  pull_request_review_comment:
    types: [created]
jobs:
  claude:
    if: contains(github.event.comment.body, '@claude') || contains(github.event.comment.body, '@Claude')
    runs-on: ubuntu-latest
    permissions:
      contents: write # needed for checkout
      pull-requests: write # needed to post comments
      issues: write # needed to post comments
    steps:
      - uses: actions/checkout@v4
        with:
          fetch-depth: 0
      - uses: anthropics/claude-code-action@beta
        with:
          anthropic_api_key: ${{ secrets.ANTHROPIC_API_KEY }}
          github_token: ${{ secrets.GITHUB_TOKEN }}
          trigger_phrase: "@claude"
          model: "claude-sonnet-4-6"
```

**`.github/workflows/claude-review.yml`** — Auto-reviews new PRs.

### Lessons learned the hard way

1. **Workflows must be on the default branch** (`main`) for `issue_comment` triggers to work
2. **Permissions must be `write`** — `read` causes 403 errors when the action tries to post comments
3. **Add `actions/checkout@v4`** before the claude-code-action — it needs the repo checked out locally
4. **Specify the model explicitly** — the action's default model (`claude-sonnet-4-20250514`) was retired and caused 404 errors
5. **Case-sensitive triggers** — `@Claude` (capital C) doesn't match `contains(..., '@claude')`. Add both cases.
6. **`ANTHROPIC_API_KEY` secret** must be added in GitHub repo settings → Secrets → Actions

### How to apply to future projects

1. Create the two workflow files on `main`
2. Add `ANTHROPIC_API_KEY` as a repository secret
3. Install the Claude GitHub App on the repo
4. Open a PR and comment `@claude review this`

---

## 7. Plugins — Community Workflow Packages

### What it is

Pre-built packages that bundle skills, agents, hooks, and commands into installable workflows. Like an app store for Claude Code workflows.

### What we installed

- **EPCC Workflow** (AWS) — Explore-Plan-Code-Commit development workflow with 12 specialized agents, slash commands (`/epcc-code`, `/epcc-explore`, `/epcc-plan`), and auto-recovery hooks

### How it maps to what we built by hand

```
Plugin = Skills + Agents + Hooks + Commands + Conventions
         ↓ bundled by someone else ↓
         One install, whole workflow
```

| Plugin feature   | What you already built manually        |
| ---------------- | -------------------------------------- |
| Bundled agents   | `.claude/agents/debugger.md`, etc.     |
| Bundled commands | `.claude/commands/start.md`, etc.      |
| Bundled skills   | `.claude/skills/vue-analyzer/SKILL.md` |
| Bundled hooks    | `.claude/settings.json` hooks section  |

### How plugins are stored

```
~/.claude/plugins/
├── known_marketplaces.json              ← registered marketplaces
├── marketplaces/
│   ├── claude-plugins-official/         ← Anthropic's marketplace
│   └── aws-claude-code-plugins/         ← AWS marketplace
└── plugin-catalog-cache.json            ← cached catalog

~/.claude/settings.json
  → "enabledPlugins": {"epcc-workflow@aws-claude-code-plugins": true}
```

### Key commands

```bash
claude plugin marketplace                              # browse
claude plugin marketplace add <url>                    # add a marketplace
claude plugin install <name>                           # install a plugin
claude plugin list                                     # list installed
claude plugin enable/disable <name>                    # toggle
claude plugin uninstall <name>                         # remove
```

### How to apply to future projects

1. Browse available plugins: `claude plugin marketplace`
2. Install what fits your stack (EPCC for structured dev, security for auditing)
3. Or build your own plugin if your team has reusable workflows

### Key insight

Plugins install globally (`~/.claude/settings.json` → `enabledPlugins`) and load from the marketplace cache. They're **not** copied into your project's `.claude/` directory. Each developer on a team needs to run `claude plugin install` themselves.

---

## 8. Agent Teams — Coordinated Multi-Agent Workflows

### What it is

Multiple Claude instances working as peers — sharing a task list, investigating independently, and messaging each other directly. Unlike subagents (parent → child → parent), teammates communicate laterally.

### How it differs from subagents

| Aspect        | Subagents                 | Agent Teams                        |
| ------------- | ------------------------- | ---------------------------------- |
| Structure     | Parent delegates to child | Peers collaborate                  |
| Communication | One-way (parent ↔ child)  | Multi-way (any ↔ any)              |
| Coordination  | Parent orchestrates       | Shared task list                   |
| Use case      | "Do this specific task"   | "Investigate from multiple angles" |

### Setup

```json
// ~/.claude/settings.json
{
  "env": {
    "CLAUDE_CODE_EXPERIMENTAL_AGENT_TEAMS": "1"
  }
}
```

### How to use

Run `claude` in the terminal and prompt:

```
Create an agent team with 3 teammates:
- Security Auditor: find vulnerabilities
- Performance Analyst: find bottlenecks
- UX Reviewer: find usability issues
Have them investigate in parallel and produce a prioritized action plan.
```

Use **Shift+Down/Up** to cycle between teammates and see their progress.

### How to apply to future projects

Best for tasks that benefit from **multiple perspectives**:

- Security + Performance + UX audit (different angles, same codebase)
- Competing hypothesis debugging (each agent tests a different theory)
- Cross-layer implementation (frontend agent + backend agent + test agent)
- Code review from multiple roles (architect + security + junior dev perspective)

### Key insight

Agent Teams is experimental (behind a feature flag). It requires the Claude CLI terminal, not the VS Code extension.

---

## 9. Worktrees — Isolated Parallel Branches

### What it is

Git worktrees create a second copy of your repo in a separate directory on a separate branch. Changes in the worktree don't affect your main working directory.

### What we did

Created a `feature/dark-mode` worktree, built a dark mode toggle prototype, committed it, then removed the worktree — all without touching the `new_features` branch.

### Commands

```bash
# Create a worktree on a new branch
git worktree add ../my-worktree -b feature/experiment

# List all worktrees
git worktree list

# Remove a worktree (keeps the branch)
git worktree remove ../my-worktree

# Delete the branch too (if discarding)
git branch -D feature/experiment
```

### Three outcomes after experimenting

| Decision                           | Action                                               |
| ---------------------------------- | ---------------------------------------------------- |
| **Keep** — merge the experiment    | `git merge feature/experiment` from your main branch |
| **Discard directory, keep branch** | `git worktree remove ../my-worktree`                 |
| **Delete everything**              | Remove worktree + `git branch -D feature/experiment` |

### How to apply to future projects

- **Prototyping** — try a risky approach without affecting your work
- **Parallel development** — work on two features simultaneously
- **Code review** — check out a PR in a worktree while keeping your branch clean
- **Claude subagents with isolation** — Claude can spawn agents in worktrees so they can't conflict with each other

### Key insight

Worktrees share the same git repo (history, remotes, objects) but have independent working directories and branches. They're much lighter than cloning the repo again.

---

## 10. Memory — Persistent Context Across Sessions

### What it is

A file-based memory system at `~/.claude/projects/<project-path>/memory/` that persists facts, feedback, and context across Claude Code sessions.

### What we saved

- **Feedback memory** — "Always document non-obvious logic changes with comments" (user preference for code comments)

### Memory types

| Type        | What to store                             | Example                                 |
| ----------- | ----------------------------------------- | --------------------------------------- |
| `user`      | User's role, preferences, knowledge level | "Senior dev, new to React"              |
| `feedback`  | Corrections and confirmed approaches      | "Don't mock the database in tests"      |
| `project`   | Ongoing work, goals, deadlines            | "Merge freeze starts March 5"           |
| `reference` | Pointers to external resources            | "Bugs tracked in Linear project INGEST" |

### Memory file format

```markdown
---
name: short-kebab-slug
description: One-line summary for relevance matching
metadata:
  type: feedback
---

Rule or fact here.
**Why:** Reason the user gave.
**How to apply:** When this guidance kicks in.
```

### How to apply to future projects

Memory is project-scoped — each project gets its own memory directory. Start saving memories when:

- A user corrects your approach → save as `feedback`
- You learn the user's role/expertise → save as `user`
- There's a deadline or initiative → save as `project`
- External tools/dashboards are referenced → save as `reference`

---

## 11. Quick Reference — File Locations

```
project-root/
├── CLAUDE.md                              ← project instructions (always loaded)
├── client/CLAUDE.md                       ← frontend-specific instructions
├── .mcp.json                              ← MCP server configuration
├── .claude/
│   ├── settings.json                      ← project hooks, permissions, env vars
│   ├── settings.local.json                ← personal overrides (gitignored)
│   ├── agents/
│   │   ├── debugger.md                    ← agent definitions
│   │   ├── code-reviewer.md
│   │   └── vue-expert.md
│   ├── skills/
│   │   ├── vue-analyzer/SKILL.md          ← skill definitions
│   │   └── backend-api-test/SKILL.md
│   └── commands/
│       ├── start.md                       ← slash commands
│       ├── stop.md
│       └── test.md
├── .github/workflows/
│   ├── claude.yml                         ← @claude mention handler
│   └── claude-review.yml                  ← auto PR review
│
~/.claude/
├── settings.json                          ← global settings (theme, model, plugins, env)
├── plugins/
│   ├── known_marketplaces.json            ← registered marketplaces
│   └── marketplaces/                      ← downloaded plugin catalogs
└── projects/<sanitized-path>/
    └── memory/                            ← persistent memory files
        ├── MEMORY.md                      ← memory index
        └── feedback_code_comments.md      ← individual memories
```

---

## 12. Applying to a New Project — Starter Checklist

When starting Claude Code on a new project, set up these in order:

### Day 1: Foundation

- [ ] Create `CLAUDE.md` with stack, quick start, key patterns, common issues
- [ ] Create `client/CLAUDE.md` (or equivalent) for frontend/backend-specific guidance
- [ ] Add `.mcp.json` with Playwright (if web app) and GitHub (if using GitHub)
- [ ] Run `claude mcp add playwright npx @playwright/mcp@latest`

### Day 2: Automation

- [ ] Install your formatter: `npm install --save-dev prettier` (or equivalent)
- [ ] Add Prettier hook to `.claude/settings.json` (PostToolUse → Edit|Write)
- [ ] Create `/start` and `/stop` commands in `.claude/commands/`

### Day 3: Specialization

- [ ] Create a `debugger` agent tailored to your stack's error patterns
- [ ] Create a `code-reviewer` agent with your project's conventions
- [ ] Create skills for repeatable analysis tasks specific to your codebase

### Day 4: CI/CD

- [ ] Set up GitHub workflows for `@claude` mentions and auto-review
- [ ] Add `ANTHROPIC_API_KEY` as a repository secret
- [ ] Install the Claude GitHub App on your repo
- [ ] Test with a real PR comment

### Day 5: Team Workflows

- [ ] Browse plugin marketplace: `claude plugin marketplace`
- [ ] Install relevant plugins (EPCC for structured dev, security for auditing)
- [ ] Enable Agent Teams if you need multi-perspective analysis
- [ ] Document your setup in the project README for team onboarding

### Ongoing

- [ ] Save `feedback` memories when corrections are given
- [ ] Update CLAUDE.md as patterns evolve
- [ ] Create new skills when you find yourself repeating instructions
- [ ] Use worktrees for experimental features

---

## Key Takeaways

1. **CLAUDE.md is the highest-ROI file** — spend time making it thorough. It's read every session.

2. **Skills vs Agents** — Skills are instructions ("how to analyze Vue components"). Agents are workers ("go analyze Vue components for me"). Use skills for guidance, agents for delegation.

3. **Hooks are fire-and-forget** — set them up once and they run silently. Start with a formatter hook, add more as you identify repetitive manual steps.

4. **MCP servers extend Claude's reach** — Playwright for browser testing, GitHub for PR management. They load at startup, so restart after adding new ones.

5. **Plugins are someone else's skills+agents+hooks bundled** — browse before building from scratch. If a community plugin covers 80% of what you need, install it and customize the rest.

6. **Worktrees protect your work** — always use them for risky experiments. The cleanup is one command.

7. **Memory bridges sessions** — save corrections and preferences so you don't re-explain context. But don't over-save — code patterns and architecture can be derived by reading the code.

8. **GitHub integration needs careful setup** — workflows must be on `main`, permissions must be `write`, checkout step is required, and the model must be explicitly specified to avoid retired model errors.
