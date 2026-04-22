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
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
echo "PROJECT_ROOT: $PROJECT_ROOT"

# Check gh CLI
if ! command -v gh >/dev/null 2>&1; then
  echo "GH_CLI: not installed"
else
  echo "GH_CLI: available"
  gh auth status 2>&1 | head -3
fi

# Detect repo identity
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner' 2>/dev/null || echo "")
if [ -n "$REPO" ]; then
  echo "REPO: $REPO"
else
  echo "REPO: unknown"
fi

# Check if already configured
if [ -f "$PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "ALREADY_CONFIGURED: true"
  echo "CONFIG_PATH: $PROJECT_ROOT/.mstack/config.yml"
else
  echo "ALREADY_CONFIGURED: false"
fi
```

If `GH_CLI` is not installed: stop and tell the user "MStack requires the GitHub CLI. Install from https://cli.github.com then re-run /mstack-setup."

If `gh auth status` shows not authenticated: stop and tell the user "gh CLI is not authenticated. Run `gh auth login` first, then re-run /mstack-setup."

---

## Step 1 — Handle re-run (if already configured)

If `ALREADY_CONFIGURED` is true, ask the user:

> .mstack/config.yml already exists for this repo. What would you like to do?
>
> A) Reconfigure from scratch (overwrite existing config)
> B) Update specific settings only
> C) Cancel — keep current config

- If **A**: continue with full setup below, overwriting at the end.
- If **B**: read the existing config, show current values, then jump directly to Step 5 (Write config) to apply user's targeted edits.
- If **C**: stop. Tell the user their existing config is unchanged.

---

## Step 2 — Detect repository

If `REPO` from the preamble is a valid `owner/repo-name` string, use it directly.

If `REPO` is "unknown" or empty, ask the user:

> I couldn't detect the GitHub repo automatically. Please enter the repo in `owner/repo-name` format (e.g. `acme/my-project`):

Store the result as `REPO` for use in the rest of the skill.

---

## Step 3 — Scan existing labels

```bash
gh label list --json name,description,color --limit 100
```

Parse the output. Identify which MStack label categories are already present and which are missing. Check for coverage across these four categories:

- **Bug/feature/enhancement/docs/question/security** (issue type labels)
- **critical/high/medium/low** (priority labels)
- **needs-triage/needs-info/confirmed/wont-fix/duplicate** (status labels)
- Any custom labels already in use

Print a summary:
- Labels present: list them
- Labels missing from MStack taxonomy: list them

---

## Step 4 — Scan repo conventions

Run each of these checks to detect how the repo is structured:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)

# Issue and PR templates
echo "=== TEMPLATES ==="
ls "$PROJECT_ROOT/.github/ISSUE_TEMPLATE/" 2>/dev/null || echo "no issue templates"
ls "$PROJECT_ROOT/.github/PULL_REQUEST_TEMPLATE*" 2>/dev/null || echo "no PR template"

# Contributing guide
echo "=== CONTRIBUTING ==="
ls "$PROJECT_ROOT/CONTRIBUTING.md" 2>/dev/null || ls "$PROJECT_ROOT/CONTRIBUTING*" 2>/dev/null || echo "no CONTRIBUTING file"

# CI workflows
echo "=== CI WORKFLOWS ==="
ls "$PROJECT_ROOT/.github/workflows/" 2>/dev/null || echo "no workflows"

# Commit style (last 20 commits)
echo "=== COMMIT STYLE (last 20) ==="
git log --oneline -20 2>/dev/null || echo "no commits"

# Test directories
echo "=== TEST DIRS ==="
for dir in test tests spec __tests__ cypress e2e; do
  [ -d "$PROJECT_ROOT/$dir" ] && echo "found: $dir"
done

# Changelog format
echo "=== CHANGELOG ==="
ls "$PROJECT_ROOT/CHANGELOG*" "$PROJECT_ROOT/CHANGES*" "$PROJECT_ROOT/HISTORY*" 2>/dev/null || echo "no changelog"

# Default/main branch
echo "=== DEFAULT BRANCH ==="
git symbolic-ref refs/remotes/origin/HEAD 2>/dev/null | sed 's|.*/||' || echo "main"
```

From these results, infer:

- **Commit style**: Conventional Commits (`feat:`, `fix:`, etc.) or freeform?
- **Test framework**: What directories exist? (jest, pytest, rspec, etc.)
- **Changelog format**: `keep-a-changelog`, `conventional`, or `none`?
- **Default branch**: `main`, `master`, or other?
- **Has CI**: yes/no

Keep a mental note of these for writing the config in Step 5.

---

## Step 5 — Configure label taxonomy

Present the user with a summary of what was found in Steps 3 and 4, then ask:

> **Label taxonomy for `{REPO}`**
>
> Current labels: {list found labels}
>
> Recommended additions to complete MStack taxonomy:
> {list missing labels with suggested colors}
>
> **Suggested additions:**
> - `bug` (#d73a4a) — Something isn't working
> - `feature` (#a2eeef) — New feature request
> - `enhancement` (#84b6eb) — Improvement to existing feature
> - `docs` (#0075ca) — Documentation changes
> - `question` (#d876e3) — Further information is requested
> - `security` (#e11d48) — Security vulnerability or concern
> - `priority: critical` (#b60205) — Needs immediate attention
> - `priority: high` (#d93f0b) — Important, address soon
> - `priority: medium` (#e4e669) — Normal priority
> - `priority: low` (#0e8a16) — Nice to have
> - `needs-triage` (#ededed) — Not yet reviewed
> - `needs-info` (#fef2c0) — Awaiting more information
> - `confirmed` (#0075ca) — Issue verified and reproducible
> - `wont-fix` (#ffffff) — Not going to be addressed
> - `duplicate` (#cfd3d7) — Already reported
>
> What would you like to do?
>
> A) Apply all recommended labels now
> B) Let me choose which labels to add
> C) Skip label setup — configure manually later

- If **A**: run `gh label create` for each missing label (see bash block below). Skip any that already exist.
- If **B**: ask the user which specific labels to create, then apply only those.
- If **C**: note that labels were skipped. Continue to Step 6.

For applying labels (run once per label to create, skipping existing):

```bash
# Example — repeat for each label to add
gh label create "needs-triage" --description "Not yet reviewed by maintainers" --color "ededed" 2>/dev/null \
  && echo "Created: needs-triage" \
  || echo "Skipped (already exists): needs-triage"
```

---

## Step 6 — Write .mstack/config.yml

Using all detected values from Steps 2–4 and user choices from Step 5, write the config file.

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
mkdir -p "$PROJECT_ROOT/.mstack/logs"
mkdir -p "$PROJECT_ROOT/.mstack/reports"
echo "Directories created: .mstack/logs, .mstack/reports"
```

Now write `.mstack/config.yml` with the detected and confirmed values. Use this format, substituting detected values where they differ from the defaults:

```yaml
repo: owner/repo-name
labels:
  categories: [bug, feature, enhancement, docs, question, security]
  priorities: [critical, high, medium, low]
  status: [needs-triage, needs-info, confirmed, wont-fix, duplicate]
triage:
  detect_duplicates: true
  detect_bots: true
  stale_days: 30
review:
  require_tests: true
  security_scan: true
  max_files_warn: 50
release:
  changelog_format: keep-a-changelog
  version_scheme: semver
  branch: main
```

Fill in the actual detected values:
- `repo`: use the `REPO` value from Step 2
- `release.branch`: use the detected default branch from Step 4
- `release.changelog_format`: use detected format (`keep-a-changelog`, `conventional`, or `none`)

---

## Step 7 — Offer to update .gitignore

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
if grep -q "\.mstack" "$PROJECT_ROOT/.gitignore" 2>/dev/null; then
  echo "GITIGNORE_STATUS: already_present"
else
  echo "GITIGNORE_STATUS: not_present"
fi
```

If `.mstack` is not already in `.gitignore`, ask the user:

> The `.mstack/` directory contains logs and reports that are typically not committed to the repo. Add `.mstack/` to `.gitignore`?
>
> A) Yes — add .mstack/ to .gitignore
> B) No — I want to commit .mstack/ to the repo

If **A**:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
echo "" >> "$PROJECT_ROOT/.gitignore"
echo "# MStack local data" >> "$PROJECT_ROOT/.gitignore"
echo ".mstack/" >> "$PROJECT_ROOT/.gitignore"
echo "Added .mstack/ to .gitignore"
```

If **B**: leave `.gitignore` unchanged.

If `.mstack` was already present in `.gitignore`: skip this step entirely and note it.

---

## Step 8 — Show summary

Print a clear summary of everything that was configured:

```
MStack setup complete for {REPO}
===========================================
Config:      .mstack/config.yml
Directories: .mstack/logs/, .mstack/reports/

Labels configured:
  {list of labels created or confirmed present}

Detected conventions:
  Default branch:    {branch}
  Changelog format:  {format}
  Commit style:      {conventional / freeform}
  Test directories:  {list or "none found"}
  CI workflows:      {yes / no}

.gitignore: {updated / already present / skipped}

Next steps:
  Run /triage to process open issues
  Run /review-prs to pre-screen pull requests
  Run /health to see a repo health snapshot
  Run /maintain for a full maintenance session
===========================================
```

If any step was skipped or had an error, note it clearly in the summary so the user knows what to address.
