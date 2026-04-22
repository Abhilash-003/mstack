# MStack development

## Commands

```bash
./setup              # install: create symlinks, config, check gh CLI
bash setup           # same, explicit bash invocation
bin/mstack-config get stale_days       # read a config value
bin/mstack-config set stale_days 60    # write a config value
bin/mstack-config list                 # show all config
```

## Testing

```bash
# Validate all skills have correct frontmatter
for f in */SKILL.md; do head -20 "$f" | grep "^name:" || echo "MISSING name: in $f"; done

# Test config read/write
MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config set stale_days 60
MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config get stale_days  # should print: 60

# Test setup script
bash setup  # should complete without errors

# Integration test: run /mstack-setup in a real repo
# (requires Claude Code session and authenticated gh CLI)
```

## Project structure

```
mstack/
├── setup                   # Install script (symlinks + config)
├── bin/
│   └── mstack-config       # YAML config read/write
├── _shared/                # Shared reference files
│   ├── detection.md        # Bot/spam detection heuristics
│   ├── gh-helpers.md       # gh CLI helper patterns
│   └── response-guidelines.md  # Tone and writing rules for /respond
├── triage/SKILL.md         # Issue categorization + duplicate/bot detection
├── review/SKILL.md         # PR pre-screening: security, tests, quality
├── respond/SKILL.md        # Tone-matched response drafting
├── health/SKILL.md         # Repo health report
├── release/SKILL.md        # Changelog + version bump + GitHub release
├── maintain/SKILL.md       # Orchestrator: chains all skills
├── setup-skill/SKILL.md    # Per-repo interactive configuration
├── templates/
│   └── responses/          # Response templates for common scenarios
│       ├── bug-confirmed.md
│       ├── duplicate.md
│       ├── needs-info.md
│       ├── stale-close.md
│       ├── welcome.md
│       └── wont-fix.md
├── SKILL.md                # Root routing skill
├── VERSION                 # Current version string
├── ARCHITECTURE.md         # Why MStack is built this way
├── CONTRIBUTING.md         # How to add skills
├── CHANGELOG.md            # Release notes
└── README.md               # User guide
```

## Skill routing

When the user's request matches a maintenance skill, invoke it:

- Wants full maintenance session → `/maintain`
- Wants to triage/label/process issues → `/triage`
- Wants to review/screen pull requests → `/review-prs`
- Wants to draft responses → `/respond`
- Wants repo health/status → `/health`
- Wants to cut a release → `/mstack-release`
- Needs to configure/setup → `/mstack-setup`

## State files

Work products go in `.mstack/` at the project root. Global config goes in `~/.mstack/`.

### Per-repo plumbing (.mstack/)

| File | Written by | Format |
|------|-----------|--------|
| `.mstack/config.yml` | `/mstack-setup` | YAML |
| `.mstack/logs/triage-YYYY-MM-DD.jsonl` | `/triage` | JSONL (append-only) |
| `.mstack/logs/review-YYYY-MM-DD.jsonl` | `/review-prs` | JSONL (append-only) |
| `.mstack/logs/maintain-YYYY-MM-DD.json` | `/maintain` | JSON (session state) |
| `.mstack/reports/YYYY-MM-DD-health.md` | `/health` | Markdown |

### Global state (~/.mstack/)

| File | Written by | Format |
|------|-----------|--------|
| `~/.mstack/config.yaml` | `bin/mstack-config` | YAML |
| `~/.mstack/.install-complete` | `./setup` | Empty marker |

## Writing SKILL templates

SKILL.md files are prompt templates read by Claude, not bash scripts. Each bash code
block runs in a separate shell. Variables do not persist between blocks.

Rules:
- Use natural language for logic and state, not shell variables between blocks
- Keep bash blocks self-contained
- Express conditionals as English decision steps
- Use AskUserQuestion for all non-trivial decisions
- Never auto-act on GitHub — every label, comment, close, or merge requires human approval
- Human approval is required before any destructive action (close issue, post comment, create tag, push)

## Config keys

| Key | Default | Used by |
|-----|---------|---------|
| `detect_duplicates` | `true` | `/triage` — semantic duplicate detection |
| `detect_bots` | `true` | `/triage`, `/review-prs` — bot/spam flagging |
| `stale_days` | `30` | `/triage`, `/health` — days before issue/PR is stale |
| `require_tests` | `true` | `/review-prs` — flag PRs without test changes |
| `security_scan` | `true` | `/review-prs` — scan for secrets, injection risks |
| `max_files_warn` | `50` | `/review-prs` — warn if PR touches more than N files |
| `changelog_format` | `keep-a-changelog` | `/mstack-release` — changelog style |
| `version_scheme` | `semver` | `/mstack-release` — version bump strategy |
| `proactive` | `true` | root SKILL.md routing — auto-suggest skills from context |
