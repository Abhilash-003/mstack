# MStack: Maintainer Stack for Claude Code

**Date:** 2026-04-13
**Status:** Draft
**Author:** MStack Contributors

---

## Problem

Open source maintainers are burning out. 60% work unpaid, 46% report burnout, and the situation is getting worse — AI-generated spam PRs and low-quality issue reports have multiplied the volume of work. GitHub Agentic Workflows offers basic 7-label triage. PR-Agent handles PR review only. No tool provides a comprehensive, end-to-end maintainer workflow.

MStack fills this gap: a Claude Code skills plugin that automates the entire maintenance lifecycle from issue triage to release shipping.

## Solution

A collection of 7 interconnected Claude Code skills (pure SKILL.md files) that chain together to handle the repetitive grunt work of open source maintenance. The maintainer stays in control — every action requires human approval before touching GitHub.

## Target Users

- Open source maintainers managing 1+ active repositories
- Project leads handling issue/PR volume they can't keep up with
- Anyone with Claude Code who maintains a GitHub repo

## Skills

### `/setup`

**Purpose:** Initialize MStack for a repository.

**What it does:**
1. Detects the current repo (or asks which repo to configure)
2. Scans existing labels, issue templates, PR templates, and contributing guidelines
3. Learns the project's conventions: naming patterns, commit style, test framework, CI setup
4. Creates `.mstack/config.yml` with:
   - Repository metadata
   - Label taxonomy (maps existing labels + suggests missing ones)
   - Triage rules (e.g., "issues mentioning 'crash' get priority:high")
   - Response templates (customizable defaults for common scenarios)
   - Review preferences (what to check in PRs)
5. Stores config locally — nothing pushed to the repo

**Config example (.mstack/config.yml):**
```yaml
repo: owner/repo-name
labels:
  categories: [bug, feature, enhancement, docs, question, security]
  priorities: [critical, high, medium, low]
  status: [needs-triage, needs-info, confirmed, wont-fix, duplicate]
triage:
  auto_label: true
  detect_duplicates: true
  detect_bots: true
  spam_threshold: 0.8
review:
  require_tests: true
  security_scan: true
  max_file_changes: 50
  bot_detection: true
release:
  changelog_format: keep-a-changelog
  version_scheme: semver
  branch: main
```

### `/triage`

**Purpose:** Process open issues that need attention.

**What it does:**
1. Fetches open issues without triage labels (or all issues, configurable)
2. For each issue, analyzes:
   - **Category**: bug, feature request, question, docs, security
   - **Priority**: based on severity, user impact, crash indicators
   - **Duplicates**: semantic similarity against existing issues (open + recently closed)
   - **Bot/spam detection**: identifies AI-generated low-quality submissions, mass-filed issues
   - **Completeness**: does it have reproduction steps? version info? expected vs actual?
   - **Sentiment**: is the reporter frustrated, blocked, or just suggesting?
3. Outputs a triage report in the terminal with recommended actions per issue
4. For each recommendation, maintainer can:
   - **Approve** — MStack applies label + posts comment via `gh`
   - **Modify** — change the label/response before applying
   - **Skip** — leave the issue untouched
   - **Batch approve** — approve all recommendations at once

**Duplicate detection approach:**
- Compares issue title + body against existing issues using keyword extraction and semantic matching
- Uses the LLM's context to understand meaning, not just string matching
- Suggests "Possible duplicate of #123" with explanation of why

**Bot/spam detection signals:**
- Account age and activity history (via GitHub API)
- Generic/templated language patterns
- Bulk submission patterns (many issues in short time)
- Known AI-generated text markers
- Mismatched issue content vs repo domain

### `/review`

**Purpose:** Pre-screen pull requests before human review.

**What it does:**
1. Fetches open PRs awaiting review (or a specific PR number)
2. For each PR, performs:
   - **Diff analysis**: reads the full diff, understands what changed and why
   - **Security scan**: identifies potential vulnerabilities (injection, hardcoded secrets, unsafe dependencies)
   - **Test coverage**: checks if new code has corresponding tests, flags untested paths
   - **Breaking changes**: detects API changes, removed exports, changed behavior
   - **Code quality**: style consistency, complexity, dead code, obvious bugs
   - **Bot detection**: AI-generated PRs that are low-quality or don't match project standards
   - **CI status**: checks if CI passed, identifies flaky test patterns
3. Produces a review summary with:
   - Risk level (low/medium/high/critical)
   - Key findings with file:line references
   - Suggested review comments (draft, not posted)
   - Recommendation: approve / request changes / close
4. Maintainer approves before any comments are posted to the PR

**What it does NOT do:**
- Auto-merge anything
- Post reviews without maintainer approval
- Reject PRs from new contributors without human review

### `/release`

**Purpose:** Automate the release process.

**What it does:**
1. Analyzes merged PRs since last release tag
2. Categorizes changes: features, bug fixes, breaking changes, docs, internal
3. Determines version bump (major/minor/patch) based on conventional commits or PR labels
4. Generates changelog entry in the project's format (Keep a Changelog, custom, etc.)
5. Presents the release plan:
   - Version: X.Y.Z
   - Changelog draft
   - PRs included with contributors credited
   - Breaking changes highlighted
6. On approval:
   - Updates CHANGELOG.md
   - Bumps VERSION file (if exists)
   - Creates git tag
   - Creates GitHub release with notes
   - Credits all contributors in release notes

### `/health`

**Purpose:** Generate a repo health report.

**What it does:**
1. **Issue health**: stale issues (no activity in N days), issues missing labels, unanswered questions
2. **PR health**: stale PRs, PRs with merge conflicts, PRs blocked on review
3. **CI health**: recent failure rate, flaky tests (pass/fail pattern), build time trends
4. **Dependency health**: security alerts from Dependabot/GitHub, outdated dependencies
5. **Community health**: response time trends, contributor activity, first-time contributor ratio
6. **Documentation health**: README freshness, broken links, missing setup instructions

Outputs a structured report with actionable items, sorted by priority. Optionally saves to `.mstack/reports/YYYY-MM-DD-health.md`.

### `/respond`

**Purpose:** Draft maintainer responses for common scenarios.

**What it does:**
1. Takes an issue or PR number (or works on current batch from `/triage`)
2. Identifies the scenario:
   - **Needs more info**: asks specific follow-up questions based on what's missing
   - **Duplicate**: polite close with link to original issue
   - **Won't fix**: explains reasoning, suggests alternatives
   - **Welcome**: first-time contributor greeting with guidance
   - **Stale close**: gentle close after inactivity, invites reopening
   - **Bug confirmed**: acknowledges, sets expectations, suggests workarounds
3. Uses response templates from `.mstack/config.yml` (customizable)
4. Adapts tone to match the project's existing communication style
5. Maintainer reviews and edits before posting

**Key principle:** Responses are always respectful and constructive. Never dismissive, never condescending. The maintainer's reputation is at stake.

### `/maintain`

**Purpose:** Full pipeline — run all skills in sequence.

**What it does:**
1. `/triage` — process new issues
2. `/review` — screen open PRs
3. `/respond` — draft responses for items needing replies
4. `/health` — generate health report
5. Checkpoint between each phase — maintainer can stop, skip, or continue
6. Summary at the end: actions taken, items skipped, follow-ups needed

**Typical use case:** Start of day, run `/maintain`, handle everything in 30 minutes instead of 2 hours.

## Architecture

### Project structure

```
mstack/
├── _shared/
│   ├── gh-helpers.md          # Shared gh CLI patterns and utilities
│   ├── templates.md           # Response template system
│   └── detection.md           # Bot/spam detection heuristics
├── setup/
│   └── SKILL.md               # /setup skill
├── triage/
│   └── SKILL.md               # /triage skill
├── review/
│   └── SKILL.md               # /review skill
├── release/
│   └── SKILL.md               # /release skill
├── health/
│   └── SKILL.md               # /health skill
├── respond/
│   └── SKILL.md               # /respond skill
├── maintain/
│   └── SKILL.md               # /maintain pipeline skill
├── templates/
│   ├── responses/
│   │   ├── needs-info.md
│   │   ├── duplicate.md
│   │   ├── wont-fix.md
│   │   ├── welcome.md
│   │   ├── stale-close.md
│   │   └── bug-confirmed.md
│   └── changelog/
│       └── keep-a-changelog.md
├── CLAUDE.md                  # Project instructions for Claude
├── ARCHITECTURE.md
├── CONTRIBUTING.md
├── README.md
├── LICENSE                    # MIT
├── setup                      # Bootstrap installation script
└── VERSION
```

### Technical decisions

**Pure SKILL.md files:** No server, no database, no dependencies beyond `gh` CLI. Each skill is a markdown file with instructions for Claude Code.

**`gh` CLI as the interface to GitHub:** Already authenticated on most developer machines. Handles auth, pagination, rate limiting. No need for PAT configuration.

**Local state in `.mstack/`:**
- `config.yml` — repo-specific configuration
- `logs/` — triage and review session logs (what was processed, what actions were taken)
- `reports/` — health check reports over time
- `cache/` — cached issue/PR data to reduce API calls within a session

**Human-in-the-loop at every step:** MStack proposes actions, the maintainer approves. Nothing is posted/labeled/closed without explicit confirmation. This is non-negotiable — the Matplotlib incident showed what happens when AI acts autonomously on repos.

### Dependencies

- **Claude Code** — runtime
- **`gh` CLI** — GitHub interaction (pre-installed on most dev machines)
- **Git** — local repo detection, tag management
- No npm/pip/cargo dependencies

### Installation

```bash
git clone --single-branch --depth 1 https://github.com/<user>/mstack.git \
  ~/.claude/skills/mstack
cd ~/.claude/skills/mstack && ./setup
```

The `setup` script:
1. Checks `gh` CLI is installed and authenticated
2. Registers skills with Claude Code
3. Prints available commands

Then run `/setup` in Claude Code inside any repo to configure MStack for that project.

## Data flow

```
Maintainer runs /triage
        │
        ▼
SKILL.md instructs Claude to:
  1. Run `gh issue list` to fetch open issues
  2. Read each issue with `gh issue view`
  3. Compare against existing issues for duplicates
  4. Analyze content for category, priority, quality
        │
        ▼
Claude presents triage report in terminal
  - Issue #42: bug, priority:high, no duplicate found
  - Issue #43: question, likely duplicate of #12
  - Issue #44: spam (bot account, generic text)
        │
        ▼
Maintainer reviews each recommendation
  - Approves #42 labeling
  - Modifies #43 response
  - Approves #44 close
        │
        ▼
Claude executes approved actions via gh CLI:
  - `gh issue edit 42 --add-label "bug,priority:high"`
  - `gh issue comment 43 --body "..."`
  - `gh issue close 44 --reason "not planned" --comment "..."`
```

## Scope boundaries

**In scope:**
- GitHub repositories (public and private)
- Issue and PR management
- Release automation
- Health monitoring
- Response drafting

**Out of scope (v1):**
- GitLab/Bitbucket support (future expansion)
- Automated CI fixes
- Dependency update PRs (Dependabot/Renovate handle this)
- Code generation or automated PR creation
- Monetization features
- Multi-repo dashboard (each repo configured independently)

## Success criteria

1. A maintainer can run `/maintain` and process their morning's issues + PRs in under 30 minutes
2. Duplicate detection catches >80% of actual duplicates
3. Bot/spam detection has <5% false positive rate on legitimate contributions
4. Zero actions taken on GitHub without explicit maintainer approval
5. Installation takes under 2 minutes
6. Works on any GitHub repo without repo-specific training data

## Risks and mitigations

| Risk | Mitigation |
|---|---|
| GitHub rate limiting on large repos | Batch API calls, cache within sessions, respect rate limits |
| False positive bot detection on new contributors | Conservative threshold, always require human confirmation |
| LLM hallucinating issue relationships | Show reasoning for duplicate detection, easy to override |
| GitHub Agentic Workflows becoming comprehensive | MStack runs locally, is fully customizable, works offline for analysis |
| Community backlash against AI in OSS | Human-in-the-loop is core design principle, transparent about AI use |

## Expansion paths (post v1)

- **Multi-repo mode**: one `/maintain` across all your repos
- **Scheduled runs**: cron-style via Claude Code scheduled tasks
- **Analytics**: trends over time (issue volume, response time, contributor growth)
- **GitLab/Bitbucket adapters**: swap `gh` for `glab`/`bb`
- **Community edition**: shared triage rules for common project types
- **IDE integration**: VS Code/JetBrains panel for triage

## Naming

**MStack** — Maintainer Stack. Short, memorable, descriptive.

Alternative: **Shepherd** — guides the flock of issues and PRs. More evocative but less immediately clear.

Open to user preference on naming.
