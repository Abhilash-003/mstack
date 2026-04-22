---
name: mstack
version: 0.1.0
description: |
  Open source maintainer automation skills for Claude Code.
  Full pipeline from morning issue flood to clean repo.
  Skills: /triage, /review-prs, /mstack-release, /health, /respond,
  /maintain (orchestrator), /mstack-setup.
allowed-tools:
  - Bash
  - Read
  - AskUserQuestion
---

## Preamble (run first)

```bash
_PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if ! git rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "WARNING: Not inside a git repository."
fi
mkdir -p "$HOME/.mstack"
_MSTACK_DIR="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
_MSTACK_CONFIG="$_MSTACK_DIR/bin/mstack-config"
_PROACTIVE=$("$_MSTACK_CONFIG" get proactive 2>/dev/null || echo "true")
echo "PROJECT_ROOT: $_PROJECT_ROOT"
echo "PROACTIVE: $_PROACTIVE"

# Check gh CLI
if command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: available"
  gh auth status 2>&1 | head -2 || echo "GH_AUTH: not authenticated"
else
  echo "GH_CLI: not installed"
fi

# Check if repo is configured
if [ -f "$_PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "unknown")
  echo "REPO: $REPO"
else
  echo "REPO_CONFIGURED: false"
fi

if [ ! -f "$HOME/.mstack/.install-complete" ]; then
  echo "NEEDS_INSTALL"
fi
```

If `GH_CLI` is not installed: tell user "MStack requires the GitHub CLI. Install from https://cli.github.com"

If `GH_AUTH` is not authenticated: tell user "gh CLI is not authenticated. Run `gh auth login` first."

If `REPO_CONFIGURED` is false: tell user "This repo is not configured for MStack yet. Run `/mstack-setup` to configure."

## MStack — Open Source Maintainer Automation

Available skills:

| Skill | What it does | When to use |
|-------|-------------|------------|
| `/maintain` | Full pipeline: triage + review + respond + health | "Morning maintenance", "process everything" |
| `/triage` | Categorize and label open issues | "Triage issues", "process new issues" |
| `/review-prs` | Pre-screen open pull requests | "Review PRs", "check open pull requests" |
| `/respond` | Draft responses for issues/PRs | "Respond to issues", "draft replies" |
| `/health` | Generate repo health report | "Repo health", "check project status" |
| `/mstack-release` | Cut a release with changelog | "Ship a release", "create release" |
| `/mstack-setup` | Configure MStack for this repo | "Setup mstack", "configure" |

## Routing

When the user's request matches a skill, invoke it using the Skill tool. Match rules:

- Wants full maintenance session → `/maintain`
- Wants to process/triage issues → `/triage`
- Wants to review/check PRs → `/review-prs`
- Wants to write/draft responses → `/respond`
- Wants repo health/status → `/health`
- Wants to cut a release → `/mstack-release`
- Needs to configure/setup → `/mstack-setup`

If unclear what the user wants, ask:

> What would you like to do?
> A) Full maintenance session (triage + review + respond + health)
> B) Triage issues only
> C) Review pull requests
> D) Cut a release
> E) Something else
