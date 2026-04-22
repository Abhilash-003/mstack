# MStack: Open Source Maintainer Automation for Claude Code

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Version](https://img.shields.io/badge/version-0.1.0-blue.svg)](CHANGELOG.md)
[![PRs Welcome](https://img.shields.io/badge/PRs-welcome-brightgreen.svg)](CONTRIBUTING.md)

Open source maintainer automation skills for Claude Code.

An OSS maintainer with 50 unread issues spends 80% of their time on grunt work: labeling, finding duplicates, screening spam PRs, drafting responses. MStack compresses that to near-zero. The judgment stays with the maintainer.

## Skills

| Skill | What it does | When to use |
|-------|-------------|------------|
| `/maintain` | Full pipeline: triage + review + respond + health | "Morning maintenance", "process everything" |
| `/triage` | Categorize issues, detect duplicates, flag bots | "Triage issues", "process the backlog" |
| `/review-prs` | Pre-screen PRs: security, tests, quality, CI | "Review PRs", "check open pull requests" |
| `/respond` | Draft tone-matched responses for issues and PRs | "Respond to issues", "draft replies" |
| `/health` | Repo health report: stale, CI, dependencies | "Repo health", "check project status" |
| `/mstack-release` | Changelog + version bump + GitHub release | "Ship a release", "cut a new version" |
| `/mstack-setup` | Configure MStack for this repo | First-time setup, label taxonomy |

## Install (30 seconds)

```bash
git clone --single-branch --depth 1 https://github.com/Abhilash-003/mstack.git ~/.claude/skills/mstack
cd ~/.claude/skills/mstack && ./setup
```

Then in Claude Code, run `/mstack-setup` inside your repo to configure it.

## Quick Start

Full maintenance session:
```
/maintain
```

Individual skills:
```
/triage
/review-prs
/respond
/health
/mstack-release
```

Specific items:
```
/review-prs 42          # review a single PR
/respond 17 #23         # draft responses for specific issues
```

## The Maintenance Pipeline

```
TRIAGE → REVIEW → RESPOND → HEALTH
  ↑          ↑        ↑        ↑
  └──────────┴────────┴────────┘
         human checkpoints
```

Every phase transition is a checkpoint. You approve the triage actions before any labels are applied. You review each PR analysis before a comment is posted. You read each draft response before it goes live. The pipeline is assistive, not autonomous.

## How It Works

Each skill is a SKILL.md file that Claude Code reads and follows. No backend, no database, no custom agents. Logs and reports live in `.mstack/` at your project root. Global config lives in `~/.mstack/`.

All GitHub actions happen through the `gh` CLI — which you're already authenticated with. No Personal Access Token, no OAuth app, no secret management.

### Architecture

- **Pure SKILL.md files** — no Express, no React, no Postgres. Claude Code IS the runtime.
- **Logs at project root** — structured JSONL in `.mstack/logs/`, health reports in `.mstack/reports/`.
- **gh CLI for everything** — already authenticated, no extra credentials needed.
- **Two-phase install** — offline bootstrap (`./setup`) + interactive per-repo setup (`/mstack-setup`).
- **Human-in-the-loop** — every label, comment, close, tag, and push requires explicit approval.

See [ARCHITECTURE.md](ARCHITECTURE.md) for the full design rationale.

## Requirements

- **Claude Code** (or any Claude Code-compatible agent)
- **gh CLI** (`brew install gh` or [cli.github.com](https://cli.github.com)) — must be authenticated
- **git**

## Project State

Internal plumbing lives in `.mstack/` at your project root.

```
your-repo/
└── .mstack/
    ├── config.yml                      # Repo identity and behavior config
    ├── logs/
    │   ├── triage-2026-04-13.jsonl     # Issue analysis (append-only)
    │   ├── review-2026-04-13.jsonl     # PR analysis (append-only)
    │   └── maintain-2026-04-13.json    # Session state
    └── reports/
        └── 2026-04-13-health.md        # Health check report
```

Add `.mstack/` to `.gitignore` to keep logs local (MStack will offer to do this during `/mstack-setup`).

## Configuration

Global config at `~/.mstack/config.yaml`:

```bash
bin/mstack-config get stale_days         # read: 30
bin/mstack-config set stale_days 60      # write
bin/mstack-config list                   # show all
```

Per-repo config at `.mstack/config.yml` — written by `/mstack-setup`, editable directly.

Key settings:

| Key | Default | Effect |
|-----|---------|--------|
| `detect_duplicates` | `true` | Semantic duplicate detection during triage |
| `detect_bots` | `true` | Bot/spam flagging in triage and PR review |
| `stale_days` | `30` | Days before an issue or PR is considered stale |
| `require_tests` | `true` | Flag PRs that change source without adding tests |
| `security_scan` | `true` | Scan PR diffs for hardcoded secrets and injection risks |
| `max_files_warn` | `50` | Warn when a PR touches more than N files |
| `changelog_format` | `keep-a-changelog` | Changelog style for `/mstack-release` |
| `version_scheme` | `semver` | Version bump strategy |

## Comparison

| | MStack | GitHub Agentic Workflows | PR-Agent | Manual |
|--|--------|--------------------------|----------|--------|
| Scope | Full maintenance pipeline | CI automation only | PR review only | Everything |
| Infrastructure | None (SKILL.md files) | GitHub Actions runners | Hosted service or self-host | None |
| Human approval | Every action | Configurable, often none | Optional | Always |
| Issue triage | Yes | No | No | Manual |
| Response drafting | Yes (tone-matched) | No | No | Manual |
| Repo health reports | Yes | No | No | Manual |
| Release automation | Yes | Partial | No | Manual |
| Auth required | gh CLI (existing) | GitHub Actions token | API key | None |
| Install | 30 seconds | Per-workflow | Complex | N/A |

## Documentation

- [README.md](README.md) — this file
- [ARCHITECTURE.md](ARCHITECTURE.md) — why MStack is built this way
- [CLAUDE.md](CLAUDE.md) — development commands, project structure, config reference
- [CONTRIBUTING.md](CONTRIBUTING.md) — how to add skills and contribute
- [CHANGELOG.md](CHANGELOG.md) — release notes

## Inspired By

- **[GStack](https://github.com/garrytan/gstack)** — engineering skills for Claude Code. MStack follows its architecture exactly.
- **[RStack](https://github.com/sunnnybala/Rstack)** — research automation skills for Claude Code. Proved the SKILL.md pattern at scale.

## License

MIT
