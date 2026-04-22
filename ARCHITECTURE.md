# Architecture

This document explains **why** MStack is built the way it is. For setup and commands, see CLAUDE.md. For contributing, see CONTRIBUTING.md.

## The core idea

MStack gives Claude Code the ability to perform open source maintenance through pure SKILL.md files. No backend, no database, no compiled binaries. Just Markdown that Claude reads and follows.

The key insight: Claude Code in 2026 is strong enough to read a GitHub issue backlog, reason about duplicates and bot signals, draft appropriate responses, and coordinate with a maintainer — when given clear, step-by-step instructions. You don't need a fleet of specialized agents. You need 7 well-written SKILL.md files.

```
Maintainer types /maintain
        │
        ▼
Claude Code reads maintain/SKILL.md
        │
        ▼
Orchestrator chains skills with human checkpoints:
        │
        ├── /triage      → categorize issues, detect duplicates, flag bots
        ├── /review-prs  → diff analysis, security scan, test coverage
        ├── /respond     → tone-matched response drafting
        └── /health      → stale issues, CI health, dependency alerts
        │
        ▼
Output: labeled issues, review comments, posted responses, health report
```

Every action requires explicit maintainer approval. Nothing is posted, labeled, or closed without a human checkpoint.

## Why no backend

The predecessor to this pattern — Ignis, the research automation platform — used Express.js + React + PostgreSQL + Supabase + a fleet of Mastra agents. It worked, but the infrastructure overhead killed iteration speed. Adding a new step meant writing an agent, registering it, adding a route, updating the frontend, testing the workflow, and deploying to Railway.

GStack proved a better way: write a Markdown file. That's the entire implementation. RStack followed GStack's architecture for research automation. MStack follows the same pattern for repository maintenance.

Every skill in MStack is a SKILL.md file with:
- **Frontmatter** declaring the skill name, version, and allowed tools
- **Preamble bash** checking environment state (gh CLI, auth, repo config)
- **Prose workflow** giving Claude step-by-step instructions

No build step. No compilation. No deploy. Edit a SKILL.md file and changes are live on the next invocation.

## Why gh CLI

MStack uses the GitHub CLI (`gh`) directly rather than wrapping the GitHub REST API. The reasons:

**Already authenticated.** Maintainers who work on GitHub repos already have `gh auth login` configured. MStack requires zero additional credentials. No Personal Access Token, no OAuth app, no secret management.

**Well-designed for scripting.** `gh issue list --json`, `gh pr diff`, `gh api` cover every operation MStack needs. The JSON output is clean and consistent.

**Same pattern as GStack.** GStack runs `git push` and `gh pr create` directly. MStack runs `gh issue edit` and `gh pr review` directly. The pattern is proven.

The only helper script is `bin/mstack-config` — a YAML config reader/writer. It exists because bash does key-value config better than prose. Everything else is direct `gh` commands.

## State management

MStack writes two kinds of state: per-repo plumbing and global config.

**Per-repo plumbing** lives in `.mstack/` at the git root:

```
my-project/
├── .mstack/
│   ├── config.yml                      # Repo identity, label taxonomy, behavior config
│   ├── logs/
│   │   ├── triage-2026-04-13.jsonl     # Issue analysis records (append-only)
│   │   ├── review-2026-04-13.jsonl     # PR analysis records (append-only)
│   │   └── maintain-2026-04-13.json    # Session state for /maintain
│   └── reports/
│       └── 2026-04-13-health.md        # Health check reports (dated)
```

JSONL files use append-only writes. Each record is one line. Skills write incrementally — the log has value even if the user cancels midway through a session. Health reports are full Markdown documents written at the end of each `/health` run.

**Global config** lives in `~/.mstack/config.yaml`, managed by `bin/mstack-config`. Flat keys only. Same `get`/`set`/`list` interface as GStack's `gstack-config`. Nothing sensitive goes here — credentials stay in `gh`'s native auth store.

## Two-phase installation

Adapted from GStack's `setup` script pattern:

1. **Bootstrap** (`./setup`, offline): verifies `gh` is installed, creates `~/.mstack/` for global state, symlinks skill directories into `~/.claude/skills/`, writes default config. Creates `.install-complete` marker.

2. **Per-repo setup** (`/mstack-setup` skill, interactive): detects the repo identity, scans existing labels, infers commit style and changelog format, writes `.mstack/config.yml`. Creates the `.mstack/` directory structure.

This separation means the install script never touches GitHub. The interactive setup happens inside Claude Code where the user can review every decision before it takes effect.

## Human-in-the-loop design

MStack is designed with one non-negotiable rule: the human approves every action that affects GitHub.

This is not a convenience feature. It is the point.

In 2023, a bot integrated with the Matplotlib project automatically closed and commented on issues without human review. When a contributor had their PR rejected and the bot posted a dismissive automated response, it escalated into a public incident that damaged the project's reputation and distressed a real person. The bot had optimized for reducing maintainer workload. It had not been designed to handle the human dynamics of open source.

MStack is built as the opposite of that system. Claude does the analysis work — reading issues, detecting duplicates, scanning diffs, finding security risks, drafting responses. The maintainer sees the analysis and makes the call. Every label applied, every comment posted, every issue closed, every tag created goes through an explicit approval step.

This makes MStack slower than a fully autonomous bot. That is the right tradeoff. The analysis is fast. The judgment stays with the maintainer.

## Skill chaining

Skills are standalone but composable. The `/maintain` orchestrator reads each sub-skill's SKILL.md file and follows it inline. Every phase transition is a human checkpoint:

```
TRIAGE → REVIEW → RESPOND → HEALTH
   ↑         ↑        ↑        ↑
   └─────────┴────────┴────────┘
          (revision loops at every checkpoint)
```

Skills discover prior context from `.mstack/` plumbing. If `/respond` finds a recent triage log, it knows which issues were already categorized. If `/health` finds prior reports, it can note trends. Each skill also degrades gracefully when prior outputs are missing — it fetches what it needs directly from GitHub.

The `/maintain` orchestrator never re-implements logic from sub-skills. It reads each skill's SKILL.md and follows it. This means improvements to `/triage` automatically improve `/maintain` — there is no separate orchestration logic to maintain.

## What MStack does NOT do

- **Auto-merge.** MStack never runs `gh pr merge`. Not with `--squash`, not with `--rebase`, not with any flag. Merge decisions belong to maintainers.
- **Auto-close.** Issues and PRs are never closed without explicit maintainer confirmation at the approval step.
- **Auto-approve.** MStack never posts `gh pr review --approve`. It posts informational comments only.
- **Auto-push.** Even in `/mstack-release`, the `git push` step always asks first.
- **Code generation.** MStack does not write code, generate patches, or modify source files. It is a maintenance and communication tool, not a code generation tool.
- **Autonomous operation.** MStack does not run on a schedule or in CI. It runs when a maintainer invokes it in a Claude Code session. The maintainer is present for every session.
