# MStack Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build MStack, a Claude Code skills plugin that automates open source maintenance — issue triage, PR review, release management, health monitoring, and response drafting — using the SKILL.md architecture pattern.

**Architecture:** Pure SKILL.md files with no backend. Claude Code IS the runtime. GitHub interaction via `gh` CLI. Local config via `bin/mstack-config` shell utility. Two-phase install: offline bootstrap (`./setup`) + interactive per-repo setup (`/mstack-setup` skill). Human-in-the-loop at every action.

**Tech Stack:** Bash (setup + config utility), Markdown SKILL.md files (all skills), `gh` CLI (GitHub API), Git (repo detection + tags)

**Reference:** Follow the SKILL.md pattern — YAML frontmatter, preamble bash block, prose workflow, AskUserQuestion checkpoints.

---

## File Map

```
mstack/
├── bin/
│   └── mstack-config              # Config utility (get/set/list)
├── _shared/
│   ├── gh-helpers.md              # Common gh CLI patterns for all skills
│   ├── detection.md               # Bot/spam detection heuristics
│   └── response-guidelines.md     # Tone and response writing rules
├── setup                          # Bootstrap installation script
├── SKILL.md                       # Root routing skill
├── setup-skill/
│   └── SKILL.md                   # /mstack-setup — per-repo configuration
├── triage/
│   └── SKILL.md                   # /triage — issue triage
├── review/
│   └── SKILL.md                   # /review-prs — PR pre-screening
├── release/
│   └── SKILL.md                   # /mstack-release — release automation
├── health/
│   └── SKILL.md                   # /health — repo health report
├── respond/
│   └── SKILL.md                   # /respond — draft maintainer responses
├── maintain/
│   └── SKILL.md                   # /maintain — full pipeline orchestrator
├── templates/
│   └── responses/
│       ├── needs-info.md
│       ├── duplicate.md
│       ├── wont-fix.md
│       ├── welcome.md
│       ├── stale-close.md
│       └── bug-confirmed.md
├── CLAUDE.md
├── ARCHITECTURE.md
├── CONTRIBUTING.md
├── README.md
├── LICENSE
├── CHANGELOG.md
└── VERSION
```

**Skill name conventions:** Skills that might conflict with common names use prefixed names:
- `/mstack-setup` (not `/setup` — too generic, may conflict)
- `/mstack-release` (not `/release` — too generic)
- `/review-prs` (not `/review` — conflicts with common skill name)
- `/triage`, `/health`, `/respond`, `/maintain` are unique enough to keep short

---

### Task 1: Project Scaffolding + Config Utility

**Files:**
- Create: `VERSION`
- Create: `LICENSE`
- Create: `bin/mstack-config`

- [ ] **Step 1: Create VERSION file**

```
0.1.0
```

Write to `VERSION`.

- [ ] **Step 2: Create MIT LICENSE**

Write the MIT license to `LICENSE` with copyright line:
```
Copyright (c) 2026 MStack Contributors
```

- [ ] **Step 3: Write bin/mstack-config**

YAML config utility with `get`, `set`, `list` interface. Config lives at `~/.mstack/config.yaml`.

```bash
#!/usr/bin/env bash
# mstack-config — read/write ~/.mstack/config.yaml
#
# Usage:
#   mstack-config get <key>          — read a config value
#   mstack-config set <key> <value>  — write a config value
#   mstack-config list               — show all config
#
# Env overrides (for testing):
#   MSTACK_STATE_DIR  — override ~/.mstack state directory
set -euo pipefail

STATE_DIR="${MSTACK_STATE_DIR:-$HOME/.mstack}"
CONFIG_FILE="$STATE_DIR/config.yaml"

CONFIG_HEADER='# mstack configuration — edit freely, changes take effect on next skill run.
# Docs: https://github.com/Abhilash-003/mstack
#
# ─── Triage ──────────────────────────────────────────────────────────
# detect_duplicates: true     # check for duplicate issues during triage
# detect_bots: true           # flag potential bot/spam submissions
# stale_days: 30              # days of inactivity before issue is stale
#
# ─── Review ──────────────────────────────────────────────────────────
# require_tests: true         # flag PRs without test changes
# security_scan: true         # check for hardcoded secrets, injection risks
# max_files_warn: 50          # warn if PR touches more than N files
#
# ─── Release ─────────────────────────────────────────────────────────
# changelog_format: keep-a-changelog   # changelog format
# version_scheme: semver               # version scheme
#
# ─── Behavior ────────────────────────────────────────────────────────
# proactive: true             # auto-suggest skills based on context
#
'

case "${1:-}" in
  get)
    KEY="${2:?Usage: mstack-config get <key>}"
    if ! printf '%s' "$KEY" | grep -qE '^[a-zA-Z0-9_]+$'; then
      echo "Error: key must contain only alphanumeric characters and underscores" >&2
      exit 1
    fi
    grep -E "^${KEY}:" "$CONFIG_FILE" 2>/dev/null | tail -1 | awk '{print $2}' | tr -d '[:space:]' || true
    ;;
  set)
    KEY="${2:?Usage: mstack-config set <key> <value>}"
    VALUE="${3:?Usage: mstack-config set <key> <value>}"
    if ! printf '%s' "$KEY" | grep -qE '^[a-zA-Z0-9_]+$'; then
      echo "Error: key must contain only alphanumeric characters and underscores" >&2
      exit 1
    fi
    mkdir -p "$STATE_DIR"
    if [ ! -f "$CONFIG_FILE" ]; then
      printf '%s' "$CONFIG_HEADER" > "$CONFIG_FILE"
    fi
    ESC_VALUE="$(printf '%s' "$VALUE" | head -1 | sed 's/[&/\]/\\&/g')"
    if grep -qE "^${KEY}:" "$CONFIG_FILE" 2>/dev/null; then
      _tmpfile="$(mktemp "${CONFIG_FILE}.XXXXXX")"
      sed "/^${KEY}:/s/.*/${KEY}: ${ESC_VALUE}/" "$CONFIG_FILE" > "$_tmpfile" && mv "$_tmpfile" "$CONFIG_FILE"
    else
      echo "${KEY}: ${VALUE}" >> "$CONFIG_FILE"
    fi
    ;;
  list)
    cat "$CONFIG_FILE" 2>/dev/null || true
    ;;
  *)
    echo "Usage: mstack-config {get|set|list} [key] [value]"
    exit 1
    ;;
esac
```

- [ ] **Step 4: Make config executable and test it**

Run:
```bash
chmod +x bin/mstack-config
MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config set detect_duplicates true
MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config get detect_duplicates
MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config list
```

Expected: `get` prints `true`. `list` shows the config file with header + the key.

- [ ] **Step 5: Commit**

```bash
git add VERSION LICENSE bin/mstack-config
git commit -m "Add project scaffolding: VERSION, LICENSE, config utility"
```

---

### Task 2: Bootstrap Setup Script

**Files:**
- Create: `setup`

- [ ] **Step 1: Write the setup script**

Bootstrap script. Checks `gh` CLI, creates state directory, symlinks skills, writes default config.

```bash
#!/usr/bin/env bash
# mstack setup — register maintainer skills with Claude Code
set -e

INSTALL_DIR="$(cd "$(dirname "$0")" && pwd)"
SOURCE_DIR="$(cd "$(dirname "$0")" && pwd -P)"
SKILLS_DIR="$(dirname "$INSTALL_DIR")"

IS_WINDOWS=0
case "$(uname -s)" in
  MINGW*|MSYS*|CYGWIN*|Windows_NT) IS_WINDOWS=1 ;;
esac

# ─── Check gh CLI ────────────────────────────────────────────
if ! command -v gh >/dev/null 2>&1; then
  echo "Error: GitHub CLI (gh) is required but not installed." >&2
  echo "Install from https://cli.github.com or your package manager." >&2
  exit 1
fi

GH_VER=$(gh --version 2>&1 | head -1)
echo "GitHub CLI: $GH_VER"

# Check auth
if ! gh auth status >/dev/null 2>&1; then
  echo ""
  echo "Warning: gh CLI is not authenticated."
  echo "Run 'gh auth login' to authenticate before using MStack skills."
  echo ""
fi

# ─── Check git ───────────────────────────────────────────────
if ! command -v git >/dev/null 2>&1; then
  echo "Error: git is required but not installed." >&2
  exit 1
fi

# ─── Create state directory ─────────────────────────────────
mkdir -p "$HOME/.mstack"

# ─── Write default config if missing ────────────────────────
MSTACK_CONFIG="$SOURCE_DIR/bin/mstack-config"
if [ ! -f "$HOME/.mstack/config.yaml" ]; then
  "$MSTACK_CONFIG" set detect_duplicates "true"
  "$MSTACK_CONFIG" set detect_bots "true"
  "$MSTACK_CONFIG" set stale_days "30"
  "$MSTACK_CONFIG" set require_tests "true"
  "$MSTACK_CONFIG" set security_scan "true"
  "$MSTACK_CONFIG" set max_files_warn "50"
  "$MSTACK_CONFIG" set changelog_format "keep-a-changelog"
  "$MSTACK_CONFIG" set version_scheme "semver"
  "$MSTACK_CONFIG" set proactive "true"
  echo "Config written to ~/.mstack/config.yaml"
fi

# ─── Symlink skill directories ──────────────────────────────
SKILL_DIRS="setup-skill triage review release health respond maintain"

echo ""
echo "Linking skills into $SKILLS_DIR ..."

for dir in $SKILL_DIRS; do
  if [ -f "$INSTALL_DIR/$dir/SKILL.md" ]; then
    SKILL_NAME=$(grep -m1 "^name:" "$INSTALL_DIR/$dir/SKILL.md" 2>/dev/null | awk '{print $2}' || echo "$dir")
    TARGET="$SKILLS_DIR/$SKILL_NAME"

    if [ "$IS_WINDOWS" -eq 1 ]; then
      if [ -d "$TARGET" ] && [ ! -L "$TARGET" ]; then
        rm -rf "$TARGET"
      fi
      ln -snf "$INSTALL_DIR/$dir" "$TARGET" 2>/dev/null || cp -rf "$INSTALL_DIR/$dir" "$TARGET"
    else
      ln -snf "$INSTALL_DIR/$dir" "$TARGET"
    fi
    echo "  /$SKILL_NAME -> $dir/"
  fi
done

# Also note root skill
if [ -f "$INSTALL_DIR/SKILL.md" ]; then
  echo "  /mstack -> ./"
fi

# ─── Mark install complete ──────────────────────────────────
touch "$HOME/.mstack/.install-complete"

echo ""
echo "=== MStack v$(cat "$SOURCE_DIR/VERSION") installed ==="
echo "Run /mstack-setup in Claude Code inside a repo to configure."
echo "Run /maintain to process issues, PRs, and repo health."
echo ""
```

- [ ] **Step 2: Make executable and verify**

Run:
```bash
chmod +x setup
```

Don't run `./setup` yet — skill files don't exist. We'll test after creating them.

- [ ] **Step 3: Commit**

```bash
git add setup
git commit -m "Add bootstrap setup script"
```

---

### Task 3: Shared Reference Docs

**Files:**
- Create: `_shared/gh-helpers.md`
- Create: `_shared/detection.md`
- Create: `_shared/response-guidelines.md`

- [ ] **Step 1: Write _shared/gh-helpers.md**

Common `gh` CLI patterns referenced by all skills.

```markdown
# gh CLI Helpers

Common patterns for MStack skills. Reference this file in SKILL.md instructions.

## Fetching Issues

```bash
# List open issues needing triage (no labels, sorted by newest)
gh issue list --state open --label "" --limit 50 --json number,title,body,author,labels,createdAt,comments,assignees

# Get full issue details
gh issue view <NUMBER> --json number,title,body,author,labels,createdAt,comments,assignees,milestone,state

# List issues with specific label
gh issue list --label "bug" --state open --json number,title,createdAt
```

## Fetching Pull Requests

```bash
# List open PRs awaiting review
gh pr list --state open --json number,title,author,createdAt,labels,reviewDecision,statusCheckRollup,additions,deletions,changedFiles,isDraft

# Get PR details with diff
gh pr view <NUMBER> --json number,title,body,author,labels,commits,files,reviewDecision,statusCheckRollup
gh pr diff <NUMBER>

# Get PR review status
gh pr checks <NUMBER>
```

## Modifying Issues

```bash
# Add labels
gh issue edit <NUMBER> --add-label "bug,priority:high"

# Add comment
gh issue comment <NUMBER> --body "comment text"

# Close with reason
gh issue close <NUMBER> --reason "not planned" --comment "Closing because..."
gh issue close <NUMBER> --reason "completed" --comment "Fixed in #<PR>"
```

## Modifying Pull Requests

```bash
# Add review comment (do NOT auto-approve or auto-merge)
gh pr review <NUMBER> --comment --body "Review comments here"

# Request changes
gh pr review <NUMBER> --request-changes --body "Changes needed..."
```

## Release Management

```bash
# List tags
git tag --sort=-v:refname | head -10

# Get commits since last tag
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "")
if [ -n "$LAST_TAG" ]; then
  git log "$LAST_TAG"..HEAD --oneline
fi

# List merged PRs since last tag
gh pr list --state merged --search "merged:>$(git log -1 --format=%ci $LAST_TAG 2>/dev/null | cut -d' ' -f1)" --json number,title,author,labels

# Create release
gh release create vX.Y.Z --title "vX.Y.Z" --notes "Release notes here"
```

## Rate Limiting

```bash
# Check current rate limit
gh api rate_limit --jq '.resources.core | "Remaining: \(.remaining)/\(.limit) Reset: \(.reset | todate)"'
```

If remaining < 100, warn the user and suggest waiting. Never burn through rate limits silently.

## Author Info

```bash
# Get info about an issue/PR author (for bot detection)
gh api "users/<USERNAME>" --jq '{login: .login, created_at: .created_at, public_repos: .public_repos, followers: .followers, type: .type}'
```
```

- [ ] **Step 2: Write _shared/detection.md**

Bot and spam detection heuristics.

```markdown
# Bot & Spam Detection Heuristics

Reference for /triage and /review skills. These are signals, not definitive indicators.
Always present findings to the maintainer for final judgment.

## Account Signals (from GitHub API)

- **Account age < 7 days** + first contribution to this repo → elevated suspicion
- **Account type is "Bot"** (GitHub API `type` field) → mark as bot
- **Zero public repos + zero followers** + generic username → elevated suspicion
- **Multiple issues filed in < 1 hour** to the same repo → bulk submission pattern

## Content Signals

- **Generic/template language**: "I noticed this issue...", "This PR improves...", overly polished language with no specific technical details
- **Mismatched domain**: issue content doesn't relate to what the repo actually does
- **No reproduction steps** for a bug report that clearly needs them
- **Copied issue text**: body is suspiciously similar to an existing issue but paraphrased
- **Excessive formatting** with minimal substance: lots of headers, bullet points, but no real information
- **AI-generated markers**: unnaturally structured text, "As an AI..." preamble (rare but happens), generic improvement suggestions

## PR-Specific Signals

- **Trivial changes dressed up**: renaming a variable, adding a comment, fixing a typo in a test — presented as a feature
- **Generated code with no tests**: large code additions with zero test coverage
- **Dependency bumps without context**: updating packages without explaining why
- **Formatting-only changes**: whitespace, import ordering across many files

## What This Is NOT

This system does NOT:
- Automatically reject any contribution
- Discriminate against new contributors (new accounts are NOT automatically bots)
- Replace human judgment

Every detection is a **signal** presented to the maintainer with reasoning. The maintainer makes the final call.
```

- [ ] **Step 3: Write _shared/response-guidelines.md**

Tone and response rules.

```markdown
# Response Guidelines

Rules for drafting maintainer responses via /respond skill.

## Core Principles

1. **Respectful always.** Even when closing spam. The maintainer's reputation is built on every interaction.
2. **Specific, not generic.** Reference the actual issue content, not boilerplate.
3. **Actionable.** If asking for more info, say exactly what's needed.
4. **Brief.** Maintainers are busy. Contributors are busy. Get to the point.
5. **Grateful.** Thank contributors for their time, especially first-timers.

## Tone Matching

Before drafting responses, read 3-5 recent comments from the maintainer in the repo.
Match their communication style:
- Formal vs casual
- Emoji usage
- Length preference
- Technical depth

## Never Do

- Use condescending language ("As I already explained...", "Obviously...")
- Be passive-aggressive ("Per my previous comment...")
- Blame the contributor ("You should have read the docs")
- Use corporate speak ("We appreciate your interest in our project")
- Include AI disclaimers ("As an AI assistant...")
- Close issues without explanation
```

- [ ] **Step 4: Commit**

```bash
git add _shared/
git commit -m "Add shared reference docs: gh helpers, detection heuristics, response guidelines"
```

---

### Task 4: Response Templates

**Files:**
- Create: `templates/responses/needs-info.md`
- Create: `templates/responses/duplicate.md`
- Create: `templates/responses/wont-fix.md`
- Create: `templates/responses/welcome.md`
- Create: `templates/responses/stale-close.md`
- Create: `templates/responses/bug-confirmed.md`

- [ ] **Step 1: Write all 6 response templates**

These are template references that the `/respond` skill reads and adapts. Placeholders use `{VARIABLE}` syntax.

**templates/responses/needs-info.md:**
```markdown
# Needs More Info

Use when an issue lacks enough detail to act on.

## Template

Thanks for reporting this, {AUTHOR}.

To help us investigate, could you provide:

{MISSING_ITEMS}

Once we have that info, we can take a closer look.
```

**templates/responses/duplicate.md:**
```markdown
# Duplicate

Use when an issue is a duplicate of an existing one.

## Template

Thanks for reporting this, {AUTHOR}. This looks like it's covered by #{ORIGINAL_ISSUE} — {ORIGINAL_TITLE}.

I'm closing this to keep discussion in one place. Feel free to add any extra context to the original issue.
```

**templates/responses/wont-fix.md:**
```markdown
# Won't Fix

Use when an issue is valid but won't be addressed.

## Template

Thanks for the suggestion, {AUTHOR}.

{REASONING}

{ALTERNATIVE_SUGGESTION}

Closing this for now, but feel free to reopen if the situation changes.
```

**templates/responses/welcome.md:**
```markdown
# Welcome First-Time Contributor

Use for first-time contributors opening a PR.

## Template

Welcome, {AUTHOR}! Thanks for your first contribution to {REPO}.

{REVIEW_SUMMARY}

{NEXT_STEPS}
```

**templates/responses/stale-close.md:**
```markdown
# Stale Close

Use for issues with no activity for an extended period.

## Template

Closing this due to inactivity ({DAYS} days without updates). If this is still relevant, feel free to reopen with updated information.
```

**templates/responses/bug-confirmed.md:**
```markdown
# Bug Confirmed

Use when a bug report is valid and reproducible.

## Template

Confirmed — thanks for the detailed report, {AUTHOR}.

{ANALYSIS}

{WORKAROUND}
```

Write each file to `templates/responses/`.

- [ ] **Step 2: Commit**

```bash
git add templates/
git commit -m "Add response templates for common maintainer scenarios"
```

---

### Task 5: Root Routing Skill (SKILL.md)

**Files:**
- Create: `SKILL.md`

- [ ] **Step 1: Write root SKILL.md**

This is the entry point that routes to sub-skills.

```markdown
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
```

- [ ] **Step 2: Commit**

```bash
git add SKILL.md
git commit -m "Add root routing skill"
```

---

### Task 6: /mstack-setup Skill

**Files:**
- Create: `setup-skill/SKILL.md`

- [ ] **Step 1: Write setup-skill/SKILL.md**

```markdown
---
name: mstack-setup
version: 0.1.0
description: |
  Configure MStack for the current repository.
  Scans repo conventions, creates .mstack/config.yml.
  Use when asked to "setup mstack", "configure mstack",
  or when any skill finds REPO_CONFIGURED: false.
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

## Preamble (run first)

```bash
_PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if ! git rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "ERROR: Not inside a git repository. MStack requires a git repo."
  exit 1
fi
mkdir -p "$HOME/.mstack"
_MSTACK_DIR="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
_MSTACK_CONFIG="$_MSTACK_DIR/bin/mstack-config"
echo "PROJECT_ROOT: $_PROJECT_ROOT"

if ! command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: not installed"
elif ! gh auth status >/dev/null 2>&1; then
  echo "GH_AUTH: not authenticated"
else
  echo "GH_CLI: ready"
  REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null || echo "")
  echo "REPO: $REPO"
fi
```

If `GH_CLI` is not installed: tell user to install from https://cli.github.com and stop.
If `GH_AUTH` is not authenticated: tell user to run `gh auth login` and stop.

**Important:** Note the `PROJECT_ROOT` and `REPO` values. All paths below are relative to PROJECT_ROOT.

## Step 1: Detect Repository

The REPO value from the preamble should contain `owner/repo-name`.

If REPO is empty, ask the user:
> Which repository should I configure? Provide in `owner/repo` format.

## Step 2: Scan Existing Labels

```bash
gh label list --json name,description,color --limit 100
```

Record the existing labels. Identify which categories are already covered (bugs, features, etc.) and which are missing.

## Step 3: Scan Repository Conventions

Detect the project's patterns by examining:

1. **Issue templates**: Check `.github/ISSUE_TEMPLATE/` for existing templates
2. **PR template**: Check `.github/PULL_REQUEST_TEMPLATE.md`
3. **Contributing guide**: Check `CONTRIBUTING.md`
4. **CI setup**: Check `.github/workflows/` for test/lint/build workflows
5. **Commit style**: Run `git log --oneline -20` to detect conventional commits vs freeform
6. **Test framework**: Look for test directories (`tests/`, `test/`, `__tests__/`, `spec/`)
7. **Changelog**: Check if `CHANGELOG.md` exists and what format it uses

Summarize findings to the user.

## Step 4: Configure Label Taxonomy

Present the current labels and suggest additions. Use AskUserQuestion:

> I found {N} existing labels. Here's what I recommend:
>
> **Already covered:** {list existing categories}
> **Suggest adding:** {list missing categories like priority:high, needs-triage, etc.}
>
> A) Apply recommended label additions
> B) Customize — let me pick which labels to add
> C) Skip — keep existing labels as-is

If A: create missing labels via `gh label create`.
If B: ask which labels they want, then create them.

## Step 5: Write Config

Create the `.mstack/` directory and write `config.yml`:

```bash
mkdir -p "$_PROJECT_ROOT/.mstack/logs" "$_PROJECT_ROOT/.mstack/reports"
```

Write `.mstack/config.yml` with the detected settings:

```yaml
# MStack configuration for {REPO}
# Generated on {DATE} — edit freely
repo: {REPO}
labels:
  categories: [{detected categories}]
  priorities: [{detected or created priority labels}]
  status: [{detected or created status labels}]
triage:
  detect_duplicates: true
  detect_bots: true
  stale_days: 30
review:
  require_tests: {detected: true if test dir exists}
  security_scan: true
  max_files_warn: 50
release:
  changelog_format: {detected format or keep-a-changelog}
  version_scheme: semver
  branch: {detected default branch}
```

## Step 6: Add .mstack to .gitignore

Check if `.mstack/` is in `.gitignore`. If not, ask:

> Should I add `.mstack/` to `.gitignore`? MStack stores local session logs and
> config there — typically you don't want to commit it.
>
> A) Yes, add to .gitignore (recommended)
> B) No, I'll manage it myself

If A: append `.mstack/` to `.gitignore`.

## Step 7: Summary

Show the configuration summary:

```
=== MStack configured for {REPO} ===
Labels:      {N} existing, {N} added
Config:      .mstack/config.yml
Triage:      duplicates={on/off}, bots={on/off}, stale={N}d
Review:      tests={on/off}, security={on/off}
Release:     {format}, {scheme}, branch={branch}

Run /maintain to start your first maintenance session.
Run /triage, /review-prs, or /health individually.
```

## Re-running Setup

If `.mstack/config.yml` already exists, show current config and ask:

> MStack is already configured for this repo. What would you like to do?
> A) Reconfigure from scratch
> B) Update specific settings
> C) Cancel
```

- [ ] **Step 2: Commit**

```bash
git add setup-skill/
git commit -m "Add /mstack-setup skill: per-repo configuration"
```

---

### Task 7: /triage Skill

**Files:**
- Create: `triage/SKILL.md`

- [ ] **Step 1: Write triage/SKILL.md**

```markdown
---
name: triage
version: 0.1.0
description: |
  Triage open issues: categorize, detect duplicates, flag bots/spam,
  suggest labels and responses. Human approves every action.
  Use when asked to "triage issues", "process issues", "label issues",
  or "handle the issue backlog".
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - AskUserQuestion
---

## Preamble (run first)

```bash
_PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if ! git rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "ERROR: Not inside a git repository."
  exit 1
fi
mkdir -p "$HOME/.mstack"
_MSTACK_DIR="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
_MSTACK_CONFIG="$_MSTACK_DIR/bin/mstack-config"
echo "PROJECT_ROOT: $_PROJECT_ROOT"

if ! command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: not installed"
elif ! gh auth status >/dev/null 2>&1; then
  echo "GH_AUTH: not authenticated"
else
  echo "GH_CLI: ready"
fi

if [ -f "$_PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}')
  echo "REPO: $REPO"
  DETECT_DUPES=$(grep "^  detect_duplicates:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "true")
  DETECT_BOTS=$(grep "^  detect_bots:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "true")
  STALE_DAYS=$(grep "^  stale_days:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "30")
  echo "DETECT_DUPLICATES: $DETECT_DUPES"
  echo "DETECT_BOTS: $DETECT_BOTS"
  echo "STALE_DAYS: $STALE_DAYS"
else
  echo "REPO_CONFIGURED: false"
fi

# Rate limit check
gh api rate_limit --jq '.resources.core | "RATE_LIMIT: \(.remaining)/\(.limit)"' 2>/dev/null || echo "RATE_LIMIT: unknown"
```

If `GH_CLI` is not installed or `GH_AUTH` is not authenticated: stop and tell user.
If `REPO_CONFIGURED` is false: tell user to run `/mstack-setup` first.

**Important:** Note PROJECT_ROOT and REPO from preamble output.

## Step 1: Fetch Open Issues

```bash
gh issue list --state open --json number,title,body,author,labels,createdAt,comments,assignees --limit 50
```

Parse the JSON output. Filter to issues that need triage — those without category labels (bug, feature, enhancement, docs, question, security).

If no issues need triage: tell user "No issues need triage. All open issues are already labeled." and stop.

Count and announce: "Found {N} issues needing triage."

## Step 2: Fetch Existing Issues for Duplicate Comparison

If DETECT_DUPLICATES is true, fetch recently active issues to compare against:

```bash
gh issue list --state all --limit 100 --json number,title,body,labels,state
```

Hold these in context for duplicate detection in Step 3.

## Step 3: Analyze Each Issue

For each issue needing triage, analyze:

1. **Read the full issue** using `gh issue view <NUMBER> --json number,title,body,author,labels,createdAt,comments`

2. **Categorize**: Determine if this is a bug, feature request, question, docs issue, security report, or other. Look for keywords, issue template markers, and content patterns.

3. **Assess priority**: Check for crash indicators, data loss, security implications, workaround availability. Assign: critical, high, medium, low.

4. **Check for duplicates** (if enabled): Compare title and body against the existing issues fetched in Step 2. Use semantic understanding, not just string matching. If a likely duplicate is found, note the original issue number and explain why.

5. **Bot/spam check** (if enabled): Read `_shared/detection.md` from the MStack skills directory for heuristics. Check the author's account:
   ```bash
   gh api "users/{AUTHOR}" --jq '{login: .login, created_at: .created_at, public_repos: .public_repos, followers: .followers, type: .type}'
   ```
   Apply the detection heuristics. Note: elevated suspicion is NOT confirmation. Present findings, don't auto-reject.

6. **Completeness check**: Does a bug report have reproduction steps? Does a feature request have a use case? What's missing?

**Write each analysis incrementally to `.mstack/logs/triage-{DATE}.jsonl`:**
```json
{"issue":42,"title":"...","category":"bug","priority":"high","duplicate_of":null,"bot_signals":[],"completeness":"missing reproduction steps","recommended_action":"label + request info","timestamp":"..."}
```

## Step 4: Present Triage Report

After analyzing all issues, present a summary table:

```
=== Triage Report: {REPO} ===
{N} issues analyzed on {DATE}

#42  [bug, high]      "App crashes on login"           → Label: bug, priority:high
#43  [question, low]  "How to configure auth?"          → Label: question, respond with docs link
#44  [duplicate]      "Login crash on mobile"           → Duplicate of #42, close
#45  [spam]           "Great project improvement idea"  → Bot account (2 days old, 0 repos), close

Actions pending your approval.
```

## Step 5: Approve Actions

Use AskUserQuestion for the batch:

> Triage complete. {N} issues analyzed. Ready to apply actions?
>
> A) Review and approve each one individually
> B) Batch approve all recommendations
> C) Approve all except flagged items (duplicates/spam)
> D) Skip — don't apply any actions now

**If A (individual review):** For each issue, show the recommendation and ask:
> Issue #{NUMBER}: "{TITLE}"
> Recommendation: {ACTION}
> {REASONING}
>
> A) Approve
> B) Modify (change label/action)
> C) Skip

On approve: execute the `gh` commands (label, comment, close as appropriate).
On modify: ask what they want to change, then execute.

**If B (batch):** Execute all recommended actions sequentially via `gh`.

**If C (approve clean):** Execute actions for non-flagged items, show flagged items for individual review.

## Step 6: Summary

After all approved actions are executed:

```
=== Triage Complete ===
Labeled:     {N} issues
Responded:   {N} issues
Closed:      {N} issues (duplicates/spam)
Skipped:     {N} issues
Log:         .mstack/logs/triage-{DATE}.jsonl
```

## Important Rules

- **Never auto-close issues without maintainer approval.** Even obvious spam requires confirmation.
- **Never post comments without approval.** Draft them, show them, let the maintainer edit.
- **Be conservative with bot detection.** False positives on legitimate first-time contributors are worse than letting a bot issue through.
- **Rate limits.** Check `RATE_LIMIT` from preamble. If remaining < 50, warn user and suggest processing fewer issues.
- **Large repos.** If there are >50 untriaged issues, process the newest 20 first and ask if the user wants to continue.
```

- [ ] **Step 2: Commit**

```bash
git add triage/
git commit -m "Add /triage skill: issue categorization, duplicate detection, bot flagging"
```

---

### Task 8: /review-prs Skill

**Files:**
- Create: `review/SKILL.md`

- [ ] **Step 1: Write review/SKILL.md**

```markdown
---
name: review-prs
version: 0.1.0
description: |
  Pre-screen open pull requests: diff analysis, security scan, test coverage,
  breaking change detection, bot detection. Human approves all review comments.
  Use when asked to "review PRs", "check pull requests", "screen PRs",
  or "pre-review open PRs".
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - AskUserQuestion
---

## Preamble (run first)

```bash
_PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if ! git rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "ERROR: Not inside a git repository."
  exit 1
fi
mkdir -p "$HOME/.mstack"
_MSTACK_DIR="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
_MSTACK_CONFIG="$_MSTACK_DIR/bin/mstack-config"
echo "PROJECT_ROOT: $_PROJECT_ROOT"

if ! command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: not installed"
elif ! gh auth status >/dev/null 2>&1; then
  echo "GH_AUTH: not authenticated"
else
  echo "GH_CLI: ready"
fi

if [ -f "$_PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}')
  echo "REPO: $REPO"
  REQ_TESTS=$(grep "^  require_tests:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "true")
  SEC_SCAN=$(grep "^  security_scan:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "true")
  MAX_FILES=$(grep "^  max_files_warn:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "50")
  echo "REQUIRE_TESTS: $REQ_TESTS"
  echo "SECURITY_SCAN: $SEC_SCAN"
  echo "MAX_FILES_WARN: $MAX_FILES"
else
  echo "REPO_CONFIGURED: false"
fi

gh api rate_limit --jq '.resources.core | "RATE_LIMIT: \(.remaining)/\(.limit)"' 2>/dev/null || echo "RATE_LIMIT: unknown"
```

If `GH_CLI` is not installed or not authenticated: stop and tell user.
If `REPO_CONFIGURED` is false: tell user to run `/mstack-setup` first.

## Step 1: Fetch Open PRs

```bash
gh pr list --state open --json number,title,author,createdAt,labels,reviewDecision,statusCheckRollup,additions,deletions,changedFiles,isDraft --limit 30
```

Filter out draft PRs unless the user specifically asks to include them. Filter out PRs that already have an approved review.

If no PRs need review: tell user "No open PRs awaiting review." and stop.

Announce: "Found {N} PRs to review."

If user provided a specific PR number as an argument, only process that PR.

## Step 2: Analyze Each PR

For each PR:

### 2a. Read the diff

```bash
gh pr diff <NUMBER>
```

Read the full diff. For very large PRs (>500 lines), read file-by-file using:
```bash
gh pr view <NUMBER> --json files --jq '.files[].path'
```
Then read the most critical files first (source code > tests > config > docs).

### 2b. Security scan (if enabled)

Check the diff for:
- **Hardcoded secrets**: API keys, tokens, passwords, connection strings (regex patterns: `(?i)(api[_-]?key|token|secret|password|passwd)\s*[:=]\s*["'][^"']+["']`)
- **SQL injection risks**: string concatenation in queries
- **Command injection**: unsanitized user input passed to shell commands
- **New dependencies**: check if `package.json`, `requirements.txt`, `Cargo.toml`, `go.mod` changed. Flag new dependencies for manual verification.
- **Permissions changes**: file permission changes, CI workflow modifications

### 2c. Test coverage check (if enabled)

Compare files changed vs test files changed:
- If source files in `src/`, `lib/`, `app/` changed but no files in `tests/`, `test/`, `__tests__/`, `spec/` changed → flag "No test changes for source modifications"
- If new functions/classes were added → flag "New code without corresponding tests"

### 2d. Breaking change detection

Look for:
- Removed or renamed public functions/methods/exports
- Changed function signatures (added required params, changed types)
- Removed API endpoints
- Changed database schema
- Modified public interfaces

### 2e. Code quality

- Large PR warning: if changedFiles > MAX_FILES_WARN, flag it
- Obvious bugs: null checks, off-by-one, resource leaks
- Dead code: unused imports, commented-out blocks
- Style inconsistency with the rest of the codebase

### 2f. Bot detection

Read `_shared/detection.md` for PR-specific signals. Check author account:
```bash
gh api "users/{AUTHOR}" --jq '{login: .login, created_at: .created_at, public_repos: .public_repos, followers: .followers, type: .type}'
```

### 2g. CI status

```bash
gh pr checks <NUMBER>
```

Note: passing, failing, or pending. If failing, identify which checks failed.

## Step 3: Generate Review Summary

For each PR, assign a **risk level**:
- **Low**: minor changes, tests pass, no security findings
- **Medium**: moderate changes, some concerns but nothing critical
- **High**: large changes, security findings, missing tests, or breaking changes
- **Critical**: hardcoded secrets, injection vulnerabilities, or CI failures

**Log to `.mstack/logs/review-{DATE}.jsonl`:**
```json
{"pr":123,"title":"...","risk":"medium","findings":["missing tests for new endpoint","1 new dependency"],"recommendation":"request_changes","timestamp":"..."}
```

## Step 4: Present Review Report

```
=== PR Review Report: {REPO} ===
{N} PRs reviewed on {DATE}

#123 [LOW]      "Fix typo in README"                 → Approve
#124 [MEDIUM]   "Add user search endpoint"           → Request changes (no tests)
#125 [HIGH]     "Refactor auth middleware"            → Request changes (breaking, 47 files)
#126 [CRITICAL] "Update config with new API key"     → Request changes (hardcoded secret!)
```

## Step 5: Approve Review Actions

Use AskUserQuestion:

> PR review complete. {N} PRs analyzed. How would you like to proceed?
>
> A) Review each PR individually (see full analysis + draft comments)
> B) See details for HIGH/CRITICAL risk PRs only
> C) Post review comments for all (after your approval of each)
> D) Skip — just wanted the report

**If A or B:** For each relevant PR, show:
- Full analysis with file:line references
- Draft review comment
- Recommendation

Ask: "Post this review comment? A) Yes B) Edit first C) Skip"

On yes: post via `gh pr review <NUMBER> --comment --body "..."` or `gh pr review <NUMBER> --request-changes --body "..."`

**Never auto-approve or auto-merge any PR.**

## Step 6: Summary

```
=== Review Complete ===
Reviewed:    {N} PRs
Commented:   {N} PRs
Skipped:     {N} PRs
Risk breakdown: {N} low, {N} medium, {N} high, {N} critical
Log:         .mstack/logs/review-{DATE}.jsonl
```

## Important Rules

- **Never approve PRs automatically.** MStack pre-screens, the maintainer approves.
- **Never merge PRs.** That is always the maintainer's decision.
- **Be fair to new contributors.** Flag quality issues but don't be discouraging.
- **Security findings are always CRITICAL or HIGH.** Never downgrade a hardcoded secret.
- **Large diffs.** For PRs with >1000 line changes, warn the user and focus on the most important files.
```

- [ ] **Step 2: Commit**

```bash
git add review/
git commit -m "Add /review-prs skill: PR pre-screening with security, tests, and quality checks"
```

---

### Task 9: /respond Skill

**Files:**
- Create: `respond/SKILL.md`

- [ ] **Step 1: Write respond/SKILL.md**

```markdown
---
name: respond
version: 0.1.0
description: |
  Draft maintainer responses for issues and PRs. Matches project tone,
  uses customizable templates, always requires human approval before posting.
  Use when asked to "respond to issues", "draft replies", "write responses",
  or "handle unanswered issues".
allowed-tools:
  - Bash
  - Read
  - Write
  - Grep
  - Glob
  - AskUserQuestion
---

## Preamble (run first)

```bash
_PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if ! git rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "ERROR: Not inside a git repository."
  exit 1
fi
mkdir -p "$HOME/.mstack"
_MSTACK_DIR="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
echo "PROJECT_ROOT: $_PROJECT_ROOT"

if ! command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: not installed"
elif ! gh auth status >/dev/null 2>&1; then
  echo "GH_AUTH: not authenticated"
else
  echo "GH_CLI: ready"
fi

if [ -f "$_PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}')
  echo "REPO: $REPO"
else
  echo "REPO_CONFIGURED: false"
fi
```

If not ready (gh missing, not authenticated, not configured): stop with appropriate message.

## Step 1: Identify What Needs Responses

If the user provided specific issue/PR numbers, use those.

Otherwise, find items needing responses:

```bash
# Issues with no maintainer response (comments from non-maintainers only)
gh issue list --state open --json number,title,author,comments,createdAt --limit 30
```

For each issue, check if there's a comment from a repo collaborator/maintainer. Issues with only external comments need responses.

Also check:
```bash
# PRs with no review
gh pr list --state open --json number,title,author,reviewDecision,createdAt --limit 20
```

PRs with `reviewDecision` of `null` or empty need attention.

Announce: "Found {N} issues and {N} PRs that need responses."

## Step 2: Learn Project Tone

Read 5 recent comments from repo maintainers to learn the communication style:

```bash
gh api "repos/{REPO}/issues/comments?sort=created&direction=desc&per_page=10" --jq '.[].body'
```

Note the tone: formal vs casual, emoji usage, length, technical depth. Read `_shared/response-guidelines.md` for rules.

## Step 3: Draft Responses

For each item needing a response:

1. Read the full issue/PR content
2. Identify the scenario (needs-info, duplicate, welcome, bug-confirmed, etc.)
3. Read the appropriate template from `templates/responses/`
4. Draft a response that:
   - Uses the template structure but adapts to the specific issue
   - Matches the project's tone from Step 2
   - Is specific to the actual content (not generic)
   - Follows `_shared/response-guidelines.md` rules

## Step 4: Present Drafts for Approval

For each drafted response, show it to the maintainer:

> **Issue #{NUMBER}: "{TITLE}"**
> Scenario: {needs-info / duplicate / welcome / etc.}
>
> Draft response:
> ---
> {FULL DRAFT TEXT}
> ---
>
> A) Post this response
> B) Edit before posting
> C) Skip this one

If B: Ask what to change, redraft, show again.
If A: Post via `gh issue comment <NUMBER> --body "..."` or `gh pr review <NUMBER> --comment --body "..."`.

## Step 5: Summary

```
=== Responses Complete ===
Drafted:    {N} responses
Posted:     {N} responses
Edited:     {N} responses (modified before posting)
Skipped:    {N} items
```

## Important Rules

- **Never post without approval.** Every response is shown to the maintainer first.
- **Never be dismissive.** Even for spam or low-quality issues, be professional.
- **No AI disclaimers.** Don't mention "As an AI..." or "This response was drafted by..."
- **Be specific.** Generic "thank you for your interest" responses are useless. Reference the actual issue content.
```

- [ ] **Step 2: Commit**

```bash
git add respond/
git commit -m "Add /respond skill: tone-matched response drafting for issues and PRs"
```

---

### Task 10: /health Skill

**Files:**
- Create: `health/SKILL.md`

- [ ] **Step 1: Write health/SKILL.md**

```markdown
---
name: health
version: 0.1.0
description: |
  Generate a repo health report: stale issues, PR status, CI health,
  dependency alerts, community metrics.
  Use when asked to "check repo health", "project status",
  "how's the repo doing", or "health check".
allowed-tools:
  - Bash
  - Read
  - Write
  - Glob
  - Grep
  - AskUserQuestion
---

## Preamble (run first)

```bash
_PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if ! git rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "ERROR: Not inside a git repository."
  exit 1
fi
mkdir -p "$HOME/.mstack"
_MSTACK_DIR="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
echo "PROJECT_ROOT: $_PROJECT_ROOT"

if ! command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: not installed"
elif ! gh auth status >/dev/null 2>&1; then
  echo "GH_AUTH: not authenticated"
else
  echo "GH_CLI: ready"
fi

if [ -f "$_PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}')
  echo "REPO: $REPO"
  STALE_DAYS=$(grep "^  stale_days:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "30")
  echo "STALE_DAYS: $STALE_DAYS"
else
  echo "REPO_CONFIGURED: false"
fi

gh api rate_limit --jq '.resources.core | "RATE_LIMIT: \(.remaining)/\(.limit)"' 2>/dev/null || echo "RATE_LIMIT: unknown"
```

If not ready: stop with appropriate message.

## Step 1: Issue Health

```bash
# All open issues with metadata
gh issue list --state open --json number,title,labels,createdAt,updatedAt,comments,assignees --limit 100
```

Analyze:
- **Stale issues**: no activity in STALE_DAYS days
- **Unlabeled issues**: missing category labels
- **Unanswered questions**: issues labeled "question" with no maintainer response
- **Unassigned**: issues with no assignee
- Total open count and age distribution

## Step 2: PR Health

```bash
gh pr list --state open --json number,title,author,createdAt,updatedAt,reviewDecision,statusCheckRollup,mergeable,isDraft --limit 50
```

Analyze:
- **Stale PRs**: no activity in STALE_DAYS days
- **Blocked PRs**: merge conflicts (`mergeable` != "MERGEABLE")
- **Awaiting review**: no review decision yet
- **Failing CI**: statusCheckRollup with failures
- **Draft PRs**: long-open drafts that may be abandoned
- Total open count

## Step 3: CI Health

```bash
# Recent workflow runs
gh run list --limit 20 --json conclusion,name,createdAt,event
```

Analyze:
- **Failure rate**: percentage of recent runs that failed
- **Flaky tests**: runs that alternate pass/fail
- **Which workflows fail most**: breakdown by workflow name

## Step 4: Dependency Health

```bash
# Dependabot alerts (if accessible)
gh api "repos/{REPO}/dependabot/alerts?state=open&per_page=20" --jq '.[].security_advisory.summary' 2>/dev/null || echo "DEPENDABOT: not accessible or no alerts"
```

If accessible, report count and severity of open alerts.
If not accessible (403), note "Dependabot alerts not accessible (may require admin permissions)."

## Step 5: Community Health

```bash
# Recent contributors (last 30 days)
git log --since="30 days ago" --format="%aN" | sort -u | wc -l

# Commit frequency
git log --since="30 days ago" --oneline | wc -l

# First-time contributors in recent PRs
gh pr list --state merged --limit 20 --json author --jq '.[].author.login' | sort -u
```

Calculate:
- Active contributors (last 30 days)
- Commit frequency
- New vs returning contributors

## Step 6: Generate Report

Write the health report to `.mstack/reports/{DATE}-health.md`:

```markdown
# Repo Health Report: {REPO}
Generated: {DATE}

## Summary
| Metric | Value | Status |
|--------|-------|--------|
| Open issues | {N} | {ok/warning/critical} |
| Stale issues | {N} | {ok/warning/critical} |
| Open PRs | {N} | {ok/warning/critical} |
| CI pass rate | {N}% | {ok/warning/critical} |
| Dependabot alerts | {N} | {ok/warning/critical} |
| Active contributors (30d) | {N} | {ok/warning/critical} |

## Action Items (by priority)
1. {CRITICAL items first}
2. {HIGH items}
3. {MEDIUM items}

## Issues
- {N} total open, {N} stale ({STALE_DAYS}d+), {N} unlabeled, {N} unanswered
- Oldest open issue: #{N} ({age})

## Pull Requests
- {N} total open, {N} stale, {N} with merge conflicts, {N} failing CI

## CI
- {N}% pass rate (last 20 runs)
- Most failing: {workflow name} ({N} failures)

## Dependencies
- {N} open Dependabot alerts

## Community
- {N} active contributors (last 30 days)
- {N} commits (last 30 days)
```

## Step 7: Present to User

Show the summary in the terminal. Then ask:

> Health report saved to `.mstack/reports/{DATE}-health.md`.
>
> Top action items:
> 1. {item}
> 2. {item}
> 3. {item}
>
> A) Address action items now (will invoke relevant skills)
> B) Done — just wanted the report

If A: Route to the appropriate skill for each action item (/triage for unlabeled issues, /respond for unanswered, etc.)
```

- [ ] **Step 2: Commit**

```bash
git add health/
git commit -m "Add /health skill: repo health report with issues, PRs, CI, deps, community"
```

---

### Task 11: /mstack-release Skill

**Files:**
- Create: `release/SKILL.md`

- [ ] **Step 1: Write release/SKILL.md**

```markdown
---
name: mstack-release
version: 0.1.0
description: |
  Cut a release: analyze merged PRs, generate changelog, bump version,
  create git tag and GitHub release. Human approves every step.
  Use when asked to "create a release", "ship a release", "cut a release",
  "bump version", or "generate changelog".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

## Preamble (run first)

```bash
_PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if ! git rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "ERROR: Not inside a git repository."
  exit 1
fi
mkdir -p "$HOME/.mstack"
_MSTACK_DIR="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
echo "PROJECT_ROOT: $_PROJECT_ROOT"

if ! command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: not installed"
elif ! gh auth status >/dev/null 2>&1; then
  echo "GH_AUTH: not authenticated"
else
  echo "GH_CLI: ready"
fi

if [ -f "$_PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}')
  CHANGELOG_FMT=$(grep "^  changelog_format:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "keep-a-changelog")
  VERSION_SCHEME=$(grep "^  version_scheme:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "semver")
  BRANCH=$(grep "^  branch:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "main")
  echo "REPO: $REPO"
  echo "CHANGELOG_FORMAT: $CHANGELOG_FMT"
  echo "VERSION_SCHEME: $VERSION_SCHEME"
  echo "BRANCH: $BRANCH"
else
  echo "REPO_CONFIGURED: false"
fi

# Current version info
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "none")
CURRENT_BRANCH=$(git branch --show-current 2>/dev/null || echo "unknown")
echo "LAST_TAG: $LAST_TAG"
echo "CURRENT_BRANCH: $CURRENT_BRANCH"
```

If not ready: stop with appropriate message.

## Step 1: Analyze Changes Since Last Release

If LAST_TAG is "none": this is the first release. All commits are included.

Otherwise:

```bash
# Commits since last tag
git log "$LAST_TAG"..HEAD --oneline

# Merged PRs since last tag
LAST_TAG_DATE=$(git log -1 --format=%ci "$LAST_TAG" 2>/dev/null | cut -d' ' -f1)
gh pr list --state merged --search "merged:>$LAST_TAG_DATE" --json number,title,author,labels --limit 100
```

## Step 2: Categorize Changes

For each merged PR / commit, categorize:

- **Features** (feat, feature, enhancement labels, or "add"/"implement" in title)
- **Bug fixes** (fix, bug labels, or "fix"/"resolve" in title)
- **Breaking changes** (breaking label, or "BREAKING" in commit/PR body)
- **Documentation** (docs label, or changes only in docs/README files)
- **Internal** (chore, refactor, ci, test labels, or internal-only changes)

## Step 3: Determine Version Bump

Based on semver and the categorized changes:
- **Major** (X.0.0): if any breaking changes
- **Minor** (x.Y.0): if new features and no breaking changes
- **Patch** (x.y.Z): if only bug fixes, docs, internal changes

If LAST_TAG is "none", suggest v0.1.0 or v1.0.0 (ask user).

Calculate the new version string.

## Step 4: Generate Changelog Entry

Read existing `CHANGELOG.md` if it exists. Follow the project's format (detected in config).

Draft a changelog entry:

```markdown
## [{NEW_VERSION}] - {DATE}

### Added
- {feature descriptions from merged PRs}

### Fixed
- {bug fix descriptions from merged PRs}

### Changed
- {other changes}

### Breaking Changes
- {breaking changes, if any}

### Contributors
- @{author1}, @{author2}, ...
```

## Step 5: Present Release Plan

Show the complete release plan:

```
=== Release Plan: {REPO} ===
Version:    {CURRENT} → {NEW}
Tag:        v{NEW}
Branch:     {BRANCH}
Changes:    {N} features, {N} fixes, {N} other
Breaking:   {yes/no}

Changelog draft:
{DRAFT}

Contributors: @{list}
```

Use AskUserQuestion:

> Does this release plan look right?
>
> A) Ship it — update changelog, tag, create GitHub release
> B) Change version (suggest different bump)
> C) Edit changelog before shipping
> D) Cancel

## Step 6: Execute Release

On approval:

1. **Update CHANGELOG.md** — prepend the new entry (or create the file if it doesn't exist)
2. **Update VERSION file** — if it exists, update to new version
3. **Commit**: `git add CHANGELOG.md VERSION && git commit -m "Release v{NEW}"`
4. **Create tag**: `git tag v{NEW}`
5. **Create GitHub release**:
   ```bash
   gh release create "v{NEW}" --title "v{NEW}" --notes "{CHANGELOG_ENTRY}"
   ```

Ask before pushing:
> Release created locally. Push tag and commit to remote?
> A) Yes, push now
> B) No, I'll push manually

If A: `git push && git push --tags`

## Step 7: Summary

```
=== Release Complete ===
Version:   v{NEW}
Tag:       v{NEW}
Changelog: CHANGELOG.md updated
Release:   https://github.com/{REPO}/releases/tag/v{NEW}
```
```

- [ ] **Step 2: Commit**

```bash
git add release/
git commit -m "Add /mstack-release skill: changelog generation, version bump, GitHub release"
```

---

### Task 12: /maintain Orchestrator Skill

**Files:**
- Create: `maintain/SKILL.md`

- [ ] **Step 1: Write maintain/SKILL.md**

```markdown
---
name: maintain
version: 0.1.0
description: |
  Full maintenance pipeline: triage issues, review PRs, draft responses,
  generate health report. Orchestrates all skills with human checkpoints.
  Use when asked to "maintain", "full maintenance", "morning maintenance",
  "process everything", or "handle the repo".
allowed-tools:
  - Bash
  - Read
  - Write
  - Edit
  - Grep
  - Glob
  - AskUserQuestion
---

## Preamble (run first)

```bash
_PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if ! git rev-parse --show-toplevel >/dev/null 2>&1; then
  echo "ERROR: Not inside a git repository."
  exit 1
fi
mkdir -p "$HOME/.mstack"
_MSTACK_DIR="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
echo "PROJECT_ROOT: $_PROJECT_ROOT"

if ! command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: not installed"
elif ! gh auth status >/dev/null 2>&1; then
  echo "GH_AUTH: not authenticated"
else
  echo "GH_CLI: ready"
fi

if [ -f "$_PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$_PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}')
  echo "REPO: $REPO"
else
  echo "REPO_CONFIGURED: false"
fi

# Quick stats for overview
OPEN_ISSUES=$(gh issue list --state open --limit 1 --json number 2>/dev/null | grep -c "number" || echo "?")
OPEN_PRS=$(gh pr list --state open --limit 1 --json number 2>/dev/null | grep -c "number" || echo "?")
echo "OPEN_ISSUES: ~$OPEN_ISSUES+"
echo "OPEN_PRS: ~$OPEN_PRS+"

gh api rate_limit --jq '.resources.core | "RATE_LIMIT: \(.remaining)/\(.limit)"' 2>/dev/null || echo "RATE_LIMIT: unknown"
```

If not ready: stop with appropriate message.

## Overview

This is the orchestrator skill. It chains /triage → /review-prs → /respond → /health with human checkpoints at each boundary.

Tell the user:

> Starting full maintenance session for **{REPO}**.
> I'll walk you through: triage → PR review → responses → health check.
> You'll have a checkpoint between each phase — skip or stop at any point.

## Phase 1: Issue Triage

Read the `/triage` skill file at the MStack skills directory (`triage/SKILL.md`) and follow it inline, skipping its preamble bash block (already run).

After completion, checkpoint:

> **Phase 1 complete: Issue Triage**
> {summary of what was done}
>
> A) Continue to Phase 2: PR Review
> B) Skip to Phase 3: Responses
> C) Skip to Phase 4: Health Check
> D) Stop here — I'm done for now

## Phase 2: PR Review

Read the `/review-prs` skill file (`review/SKILL.md`) and follow it inline, skipping its preamble.

After completion, checkpoint:

> **Phase 2 complete: PR Review**
> {summary}
>
> A) Continue to Phase 3: Responses
> B) Skip to Phase 4: Health Check
> C) Stop here

## Phase 3: Response Drafting

Read the `/respond` skill file (`respond/SKILL.md`) and follow it inline, skipping its preamble.

After completion, checkpoint:

> **Phase 3 complete: Responses**
> {summary}
>
> A) Continue to Phase 4: Health Check
> B) Stop here

## Phase 4: Health Check

Read the `/health` skill file (`health/SKILL.md`) and follow it inline, skipping its preamble.

## Final: Session Summary

```
=== Maintenance Session Complete: {REPO} ===
Date: {DATE}

Triage:    {N} issues processed, {N} labeled, {N} closed
Review:    {N} PRs reviewed, {N} commented
Responses: {N} drafted, {N} posted
Health:    Report at .mstack/reports/{DATE}-health.md

Top follow-ups:
1. {item}
2. {item}
3. {item}
```

## Important Rules

- This skill READS other skill files and follows them inline. It does NOT re-implement their logic.
- Every phase boundary is a human checkpoint. The user can stop, skip, or continue.
- If any phase fails, show the error, offer: Retry / Skip / Stop.
- State is written to disk at each phase, so nothing is lost if the session ends.
- If the user runs `/maintain` again, it starts fresh (no resume from previous session).
```

- [ ] **Step 2: Commit**

```bash
git add maintain/
git commit -m "Add /maintain skill: full pipeline orchestrator with phase checkpoints"
```

---

### Task 13: Project Documentation

**Files:**
- Create: `CLAUDE.md`
- Create: `ARCHITECTURE.md`
- Create: `CONTRIBUTING.md`
- Create: `README.md`
- Create: `CHANGELOG.md`

- [ ] **Step 1: Write CLAUDE.md**

Development reference for Claude Code (and human contributors).

```markdown
# MStack development

## Commands

```bash
./setup                          # install: create symlinks, config
bin/mstack-config get stale_days # read a config value
bin/mstack-config set stale_days 14  # write a config value
bin/mstack-config list           # show all config
```

## Testing

```bash
# Validate all skills have correct frontmatter
for f in */SKILL.md; do head -20 "$f" | grep "^name:" || echo "MISSING name: in $f"; done

# Test config read/write
MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config set detect_bots true
MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config get detect_bots  # should print: true

# Test setup script
bash setup  # should complete without errors (needs gh CLI)

# Integration test: run /triage on a real repo
# (requires Claude Code session + gh auth)
```

## Project structure

```
mstack/
├── setup                      # Install script (symlinks + config)
├── bin/                       # CLI utilities
│   └── mstack-config          # YAML config read/write
├── _shared/                   # Shared reference files
│   ├── gh-helpers.md          # Common gh CLI patterns
│   ├── detection.md           # Bot/spam detection heuristics
│   └── response-guidelines.md # Tone and response rules
├── triage/SKILL.md            # Issue triage skill
├── review/SKILL.md            # PR review skill
├── respond/SKILL.md           # Response drafting skill
├── health/SKILL.md            # Repo health report skill
├── release/SKILL.md           # Release automation skill
├── maintain/SKILL.md          # Orchestrator: chains all skills
├── setup-skill/SKILL.md       # Per-repo configuration
├── templates/responses/       # Response templates
├── SKILL.md                   # Root routing skill
├── ARCHITECTURE.md            # Why MStack is built this way
├── CONTRIBUTING.md            # How to contribute
├── CHANGELOG.md               # Release notes
└── README.md                  # User guide
```

## Skill routing

When the user's request matches a skill, invoke it:

- Full maintenance session → `/maintain`
- Triage/label issues → `/triage`
- Review/check PRs → `/review-prs`
- Draft responses → `/respond`
- Repo health → `/health`
- Cut a release → `/mstack-release`
- Configure repo → `/mstack-setup`

## State files

Per-repo plumbing lives in `.mstack/` at the project root.

| File | Written by | Format |
|------|-----------|--------|
| `.mstack/config.yml` | /mstack-setup | YAML |
| `.mstack/logs/triage-{DATE}.jsonl` | /triage | JSONL |
| `.mstack/logs/review-{DATE}.jsonl` | /review-prs | JSONL |
| `.mstack/reports/{DATE}-health.md` | /health | Markdown |

## Writing SKILL templates

SKILL.md files are prompt templates read by Claude, not bash scripts. Each bash code
block runs in a separate shell. Variables do not persist between blocks.

Rules:
- Use natural language for logic and state, not shell variables between blocks
- Keep bash blocks self-contained
- Express conditionals as English decision steps
- Use AskUserQuestion for all non-trivial decisions
- Always require human approval before modifying GitHub (labels, comments, closes)

## Config keys

| Key | Default | Used by |
|-----|---------|---------|
| `detect_duplicates` | `true` | /triage |
| `detect_bots` | `true` | /triage, /review-prs |
| `stale_days` | `30` | /triage, /health |
| `require_tests` | `true` | /review-prs |
| `security_scan` | `true` | /review-prs |
| `max_files_warn` | `50` | /review-prs |
| `changelog_format` | `keep-a-changelog` | /mstack-release |
| `version_scheme` | `semver` | /mstack-release |
| `proactive` | `true` | root SKILL.md routing |
```

- [ ] **Step 2: Write ARCHITECTURE.md**

Explain WHY, not just WHAT.

Write `ARCHITECTURE.md` explaining:
- The core idea (Claude Code as runtime, SKILL.md files)
- Why no backend (SKILL.md pattern is proven and simple)
- Why `gh` CLI (no PAT, already authenticated)
- State management (.mstack/ plumbing)
- Two-phase install
- Human-in-the-loop design (and the Matplotlib incident context)
- What MStack does NOT do (no auto-merge, no auto-close, no auto-approve)

- [ ] **Step 3: Write CONTRIBUTING.md**

Cover:
- Quick start (clone + setup)
- How skills work (frontmatter + preamble + prose)
- Adding a new skill
- Editing existing skills
- Conventions (follow existing patterns, human checkpoints, never auto-act)
- Testing (frontmatter validation, config tests, integration tests)
- Commit style
- What NOT to change

- [ ] **Step 4: Write README.md**

The most important file for open source exposure. Structure:
- Badges (license, version, PRs welcome)
- One-line description
- Problem statement (maintainer burnout + AI spam)
- Skills table
- Install instructions (30 seconds)
- Quick start examples
- The maintenance pipeline diagram
- How it works
- Architecture summary
- Requirements (Claude Code, gh CLI, git)
- Project state diagram
- Configuration
- Comparison table (vs GitHub Agentic Workflows, PR-Agent, manual)
- Documentation links
- License
- License

- [ ] **Step 5: Write CHANGELOG.md**

```markdown
# Changelog

All notable changes to MStack will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/).

## [0.1.0] - 2026-04-13

### Added
- `/triage` skill: issue categorization, duplicate detection, bot flagging
- `/review-prs` skill: PR pre-screening with security, tests, quality checks
- `/respond` skill: tone-matched response drafting
- `/health` skill: repo health report
- `/mstack-release` skill: changelog + version bump + GitHub release
- `/maintain` skill: full pipeline orchestrator
- `/mstack-setup` skill: per-repo configuration
- `bin/mstack-config` utility for config management
- Response templates for common scenarios
- Bot/spam detection heuristics
- Shared `gh` CLI helpers
```

- [ ] **Step 6: Commit**

```bash
git add CLAUDE.md ARCHITECTURE.md CONTRIBUTING.md README.md CHANGELOG.md
git commit -m "Add project documentation: README, ARCHITECTURE, CONTRIBUTING, CLAUDE.md, CHANGELOG"
```

---

### Task 14: GitHub Repository Setup

**Files:**
- Create: `.github/ISSUE_TEMPLATE/bug_report.md`
- Create: `.github/ISSUE_TEMPLATE/feature_request.md`
- Create: `.github/PULL_REQUEST_TEMPLATE.md`
- Create: `.gitignore`

- [ ] **Step 1: Write .gitignore**

```
# MStack local state (per-repo)
.mstack/

# OS
.DS_Store
Thumbs.db

# Editor
*.swp
*.swo
*~
.vscode/
.idea/
```

- [ ] **Step 2: Write bug report template**

`.github/ISSUE_TEMPLATE/bug_report.md`:
```markdown
---
name: Bug Report
about: Report a bug in an MStack skill
labels: bug
---

**Skill:** (e.g., /triage, /review-prs, /maintain)

**What happened:**

**What I expected:**

**Steps to reproduce:**
1.
2.
3.

**Environment:**
- OS:
- Claude Code version:
- gh CLI version: (`gh --version`)
```

- [ ] **Step 3: Write feature request template**

`.github/ISSUE_TEMPLATE/feature_request.md`:
```markdown
---
name: Feature Request
about: Suggest a new feature or skill improvement
labels: enhancement
---

**Skill:** (e.g., /triage, or "new skill")

**Problem:** What maintenance pain point would this solve?

**Proposed solution:**

**Alternatives considered:**
```

- [ ] **Step 4: Write PR template**

`.github/PULL_REQUEST_TEMPLATE.md`:
```markdown
## What

## Why

## Testing
- [ ] Frontmatter validation passes (`for f in */SKILL.md; do head -20 "$f" | grep "^name:" || echo "MISSING: $f"; done`)
- [ ] Config utility tests pass
- [ ] Tested skill manually in Claude Code
```

- [ ] **Step 5: Commit**

```bash
git add .gitignore .github/
git commit -m "Add GitHub templates and gitignore"
```

---

### Task 15: Final Validation + Setup Test

- [ ] **Step 1: Validate all skill frontmatter**

Run:
```bash
for f in */SKILL.md SKILL.md; do echo "--- $f ---"; head -20 "$f" | grep "^name:" || echo "MISSING name: in $f"; done
```

Expected: every SKILL.md has a `name:` field. Fix any missing.

- [ ] **Step 2: Test config utility**

Run:
```bash
MSTACK_STATE_DIR=/tmp/mstack-test-final bin/mstack-config set detect_bots true
[ "$(MSTACK_STATE_DIR=/tmp/mstack-test-final bin/mstack-config get detect_bots)" = "true" ] && echo "PASS" || echo "FAIL"
MSTACK_STATE_DIR=/tmp/mstack-test-final bin/mstack-config set stale_days 14
[ "$(MSTACK_STATE_DIR=/tmp/mstack-test-final bin/mstack-config get stale_days)" = "14" ] && echo "PASS" || echo "FAIL"
rm -rf /tmp/mstack-test-final
```

Expected: both PASS.

- [ ] **Step 3: Test setup script (dry run)**

Run:
```bash
bash -n setup  # syntax check only
echo $?
```

Expected: exit code 0 (no syntax errors).

- [ ] **Step 4: Verify file structure matches spec**

Run:
```bash
find . -type f -not -path './.git/*' | sort
```

Compare against the file map at the top of this plan. Ensure all expected files exist.

- [ ] **Step 5: Commit any fixes**

If any validation revealed issues, fix them and commit:
```bash
git add -A
git commit -m "Fix validation issues from final check"
```

If no issues, skip this step.

- [ ] **Step 6: Final commit — tag v0.1.0**

```bash
git tag v0.1.0
```

Do NOT push — wait for user to create the GitHub repo and set up the remote.

---

## Plan Self-Review

**Spec coverage check:**
- [x] `/setup` skill (Task 6)
- [x] `/triage` skill (Task 7)
- [x] `/review` skill (Task 8)
- [x] `/release` skill (Task 11)
- [x] `/health` skill (Task 10)
- [x] `/respond` skill (Task 9)
- [x] `/maintain` orchestrator (Task 12)
- [x] Config utility (Task 1)
- [x] Bootstrap setup script (Task 2)
- [x] Shared docs (Task 3)
- [x] Response templates (Task 4)
- [x] Root routing skill (Task 5)
- [x] Project documentation (Task 13)
- [x] GitHub repo setup (Task 14)
- [x] Validation (Task 15)

**Placeholder scan:** No TBDs, TODOs, or "fill in later" found.

**Type/name consistency:**
- Skill names: `mstack`, `mstack-setup`, `triage`, `review-prs`, `mstack-release`, `health`, `respond`, `maintain` — consistent across root SKILL.md routing table, CLAUDE.md, README references, and individual SKILL.md frontmatter.
- Config keys: `detect_duplicates`, `detect_bots`, `stale_days`, `require_tests`, `security_scan`, `max_files_warn`, `changelog_format`, `version_scheme`, `proactive` — consistent across config utility defaults, CLAUDE.md table, and skill preamble reads.
- Directory names match throughout: `setup-skill/`, `triage/`, `review/`, `release/`, `health/`, `respond/`, `maintain/`.
