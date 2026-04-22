---
name: health
version: 0.1.0
description: |
  Generate a repo health report: stale issues, PR status, CI health,
  dependency alerts, and community metrics. Writes a dated report to
  .mstack/reports/ and presents a summary with prioritized action items.
  Use when asked for "repo health", "project status", "health check",
  "stale issues", or "check the repo".
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

# Check rate limit
if command -v gh >/dev/null 2>&1; then
  RATE_REMAINING=$(gh api rate_limit --jq '.rate.remaining' 2>/dev/null || echo "unknown")
  echo "RATE_REMAINING: $RATE_REMAINING"
fi

# Check if repo is configured
if [ -f "$PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "unknown")
  STALE_DAYS=$(grep "stale_days:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "30")
  echo "REPO: $REPO"
  echo "STALE_DAYS: $STALE_DAYS"
else
  echo "REPO_CONFIGURED: false"
fi
```

If `GH_CLI` is not installed: stop and tell the user "MStack requires the GitHub CLI. Install from https://cli.github.com then re-run /health."

If `gh auth status` shows not authenticated: stop and tell the user "gh CLI is not authenticated. Run `gh auth login` first, then re-run /health."

If `REPO_CONFIGURED` is false: stop and tell the user "This repo is not configured for MStack yet. Run `/mstack-setup` first."

If `RATE_REMAINING` is a number less than 50: warn the user "GitHub API rate limit is low ({RATE_REMAINING} requests remaining). Health report may be throttled. Continuing anyway — watch for errors."

---

## Step 1 — Issue health

```bash
gh issue list --state open \
  --json number,title,author,labels,createdAt,updatedAt,comments,assignees \
  --limit 100
```

Parse the JSON output. For each issue, calculate `days_since_update` as the number of days since `updatedAt`. Use `STALE_DAYS` as the staleness threshold.

Analyze and count:

- **Stale**: issues with `days_since_update >= STALE_DAYS`
- **Unlabeled**: issues with an empty `labels` array
- **Unanswered questions**: issues where the title or body contains a `?` (question), and `comments` is an empty array — no one has replied
- **Unassigned**: issues with an empty `assignees` array

For age distribution, bucket open issues by age:
- 0–7 days: `fresh`
- 8–30 days: `recent`
- 31–90 days: `aging`
- 90+ days: `old`

Record counts for each bucket.

Store:
- `ISSUE_TOTAL` — total open issues
- `ISSUE_STALE` — count of stale issues
- `ISSUE_UNLABELED` — count of unlabeled issues
- `ISSUE_UNANSWERED` — count of unanswered questions
- `ISSUE_UNASSIGNED` — count of unassigned issues
- `ISSUE_AGE_BUCKETS` — counts for each age bucket

---

## Step 2 — PR health

```bash
gh pr list --state open \
  --json number,title,author,isDraft,reviewDecision,createdAt,updatedAt,mergeable,labels,reviews,statusCheckRollup \
  --limit 50
```

Parse the JSON output. For each PR, calculate `days_since_update` using `updatedAt`.

Analyze and count:

- **Stale**: PRs with `days_since_update >= STALE_DAYS`
- **Blocked (merge conflicts)**: PRs where `mergeable` is `"CONFLICTING"`
- **Awaiting review**: PRs where `isDraft` is `false` and `reviewDecision` is `null` or `""`
- **Failing CI**: PRs where `statusCheckRollup` contains any entry with `conclusion` of `"FAILURE"` or `"TIMED_OUT"`
- **Abandoned drafts**: draft PRs (`isDraft` is `true`) that are stale (`days_since_update >= STALE_DAYS`)

Store:
- `PR_TOTAL` — total open PRs
- `PR_STALE` — count of stale PRs
- `PR_BLOCKED` — count with merge conflicts
- `PR_AWAITING_REVIEW` — count awaiting review
- `PR_FAILING_CI` — count with failing CI
- `PR_ABANDONED_DRAFTS` — count of abandoned drafts

---

## Step 3 — CI health

```bash
gh run list --limit 20 \
  --json conclusion,name,createdAt,event
```

Parse the JSON output. Calculate:

- **Failure rate**: percentage of runs where `conclusion` is `"failure"` (excluding in-progress runs where `conclusion` is `null`)
- **Most-failing workflows**: group runs by `name`, count failures per workflow, list the top 3 most-failing
- **Flaky tests**: workflows that appear in both `"success"` and `"failure"` conclusions within the last 20 runs — these alternate between pass and fail

Classify CI health:
- `ok` — failure rate < 10%
- `warning` — failure rate 10–30%
- `critical` — failure rate > 30%

Store:
- `CI_FAILURE_RATE` — as a percentage
- `CI_TOP_FAILING` — list of up to 3 workflow names with failure counts
- `CI_FLAKY` — list of workflow names identified as flaky
- `CI_STATUS` — `ok`, `warning`, or `critical`

---

## Step 4 — Dependency health

```bash
gh api "repos/{REPO}/dependabot/alerts?state=open&per_page=20" 2>/dev/null || echo "DEPENDABOT_STATUS: not_accessible"
```

Replace `{REPO}` with the repo value from config.

If the command returns a `403` or outputs `DEPENDABOT_STATUS: not_accessible`: record `DEPS_ACCESSIBLE` as `false` and note "Dependabot alerts not accessible (requires admin access or Dependabot not enabled on this repo)." Do not treat this as an error — continue to Step 5.

If accessible, parse the alerts and count by severity:
- `critical`
- `high`
- `medium`
- `low`

Classify dependency health:
- `ok` — zero open alerts
- `warning` — only medium/low alerts, no high or critical
- `critical` — any high or critical alerts

Store:
- `DEPS_ACCESSIBLE` — `true` or `false`
- `DEPS_CRITICAL` — count of critical-severity alerts
- `DEPS_HIGH` — count of high-severity alerts
- `DEPS_MEDIUM` — count of medium-severity alerts
- `DEPS_LOW` — count of low-severity alerts
- `DEPS_STATUS` — `ok`, `warning`, `critical`, or `unknown` (if not accessible)

---

## Step 5 — Community health

```bash
# Contributors and commit activity in the last 30 days
git log --since="30 days ago" --format="%ae" | sort | uniq -c | sort -rn
```

Parse the output to get a list of unique contributor email addresses and commit counts. Count unique contributors as `CONTRIBUTORS_30D`. Count total commits in the window as `COMMITS_30D`.

Then fetch recent merged PR authors:

```bash
gh pr list --state merged --limit 20 \
  --json number,author,mergedAt \
  --jq '.[] | select(.mergedAt != null) | {number: .number, author: .author.login, mergedAt: .mergedAt}'
```

Filter to PRs merged within the last 30 days. Count unique authors as `MERGED_PR_AUTHORS_30D`.

Classify community health:
- `ok` — 3 or more contributors in the last 30 days
- `warning` — 1–2 contributors in the last 30 days
- `critical` — 0 contributors in the last 30 days (no activity)

Store:
- `CONTRIBUTORS_30D` — unique contributor count (by email)
- `COMMITS_30D` — total commits in last 30 days
- `MERGED_PR_AUTHORS_30D` — unique merged PR authors in last 30 days
- `COMMUNITY_STATUS` — `ok`, `warning`, or `critical`

---

## Step 6 — Generate report

Determine overall health status:
- `ok` — all individual statuses are `ok`
- `warning` — at least one status is `warning` and none is `critical`
- `critical` — at least one status is `critical`

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
DATE=$(date +%Y-%m-%d)
mkdir -p "$PROJECT_ROOT/.mstack/reports"
REPORT_FILE="$PROJECT_ROOT/.mstack/reports/$DATE-health.md"
```

Write the report to `$REPORT_FILE`. The report must contain the following sections:

---

### Report format

```markdown
# Repo Health Report — {DATE}

**Repo:** {REPO}
**Generated:** {DATE} {TIME}
**Overall status:** {ok | warning | critical}

---

## Summary

| Metric                   | Value         | Status    |
|--------------------------|---------------|-----------|
| Open issues              | {N}           | {ok/warning/critical} |
| Stale issues             | {N}           | {ok/warning/critical} |
| Unlabeled issues         | {N}           | {ok/warning/critical} |
| Unanswered questions     | {N}           | {ok/warning/critical} |
| Open PRs                 | {N}           | {ok/warning/critical} |
| Stale PRs                | {N}           | {ok/warning/critical} |
| PRs with conflicts       | {N}           | {ok/warning/critical} |
| PRs awaiting review      | {N}           | {ok/warning/critical} |
| CI failure rate          | {N}%          | {ok/warning/critical} |
| Dependency alerts        | {N} ({breakdown}) | {ok/warning/critical/unknown} |
| Contributors (30d)       | {N}           | {ok/warning/critical} |
| Commits (30d)            | {N}           | —         |

---

## Action Items

### Critical
{List action items for critical-status metrics, if any. Each item: "- Fix: {description}"}

### Warning
{List action items for warning-status metrics, if any. Each item: "- Address: {description}"}

### Informational
{Positive notes or low-priority observations}

---

## Issues

- **Total open:** {ISSUE_TOTAL}
- **Stale (>{STALE_DAYS} days without update):** {ISSUE_STALE}
- **Unlabeled:** {ISSUE_UNLABELED}
- **Unanswered questions:** {ISSUE_UNANSWERED}
- **Unassigned:** {ISSUE_UNASSIGNED}

Age distribution:
- 0–7 days (fresh): {N}
- 8–30 days (recent): {N}
- 31–90 days (aging): {N}
- 90+ days (old): {N}

---

## Pull Requests

- **Total open:** {PR_TOTAL}
- **Stale (>{STALE_DAYS} days without update):** {PR_STALE}
- **Merge conflicts:** {PR_BLOCKED}
- **Awaiting review:** {PR_AWAITING_REVIEW}
- **Failing CI:** {PR_FAILING_CI}
- **Abandoned drafts:** {PR_ABANDONED_DRAFTS}

---

## CI

- **Failure rate (last 20 runs):** {CI_FAILURE_RATE}%
- **Status:** {CI_STATUS}
- **Most-failing workflows:** {CI_TOP_FAILING}
- **Flaky workflows:** {CI_FLAKY}

---

## Dependencies

{If DEPS_ACCESSIBLE is false:}
Dependabot alerts not accessible. This may mean Dependabot is not enabled or this token lacks admin access.

{If DEPS_ACCESSIBLE is true:}
- **Critical:** {DEPS_CRITICAL}
- **High:** {DEPS_HIGH}
- **Medium:** {DEPS_MEDIUM}
- **Low:** {DEPS_LOW}

---

## Community

- **Unique contributors (last 30 days):** {CONTRIBUTORS_30D}
- **Total commits (last 30 days):** {COMMITS_30D}
- **Unique merged PR authors (last 30 days):** {MERGED_PR_AUTHORS_30D}
```

---

After writing the file, confirm:

```bash
echo "Report written to $REPORT_FILE"
```

---

## Step 7 — Present to user

Display the summary table from the report in the terminal. Format it clearly with the metric, value, and status columns. Use plain text — no markdown rendering assumed.

Then display the top 3 action items by priority (critical first, then warning). If there are fewer than 3 actionable items, show all of them. If there are none, note "No critical or warning items found — repo health looks good."

Finally, ask the user:

> Health report saved to `.mstack/reports/{DATE}-health.md`
>
> What would you like to do next?
>
> A) Address action items now — route to relevant skills (triage stale issues, review PRs, fix CI, etc.)
> B) Done — no further action needed

If the user chooses **A**, determine which skills are most relevant based on the critical and warning items:

- Stale issues, unlabeled issues, unanswered questions → suggest running `/triage`
- Stale PRs, PRs awaiting review, PRs with failing CI → suggest running `/review-prs`
- Issues or PRs needing responses → suggest running `/respond`
- Dependency alerts (if accessible and flagged) → note that dependency alerts require manual review in the GitHub Security tab

Present the recommended next steps clearly, then stop. Do not automatically invoke other skills — let the user run them.

If the user chooses **B**, print:

```
Health check complete. Report: .mstack/reports/{DATE}-health.md
```

...and stop.

---

## Status thresholds

Use these thresholds when classifying individual metrics in the summary table:

| Metric                 | ok          | warning         | critical         |
|------------------------|-------------|-----------------|------------------|
| Stale issues           | 0           | 1–5             | 6+               |
| Unlabeled issues       | 0–2         | 3–10            | 11+              |
| Unanswered questions   | 0–1         | 2–5             | 6+               |
| Stale PRs              | 0           | 1–3             | 4+               |
| PRs with conflicts     | 0           | 1–2             | 3+               |
| PRs awaiting review    | 0–2         | 3–5             | 6+               |
| CI failure rate        | < 10%       | 10–30%          | > 30%            |
| Dependency alerts      | 0           | medium/low only | any high/critical|
| Contributors (30d)     | 3+          | 1–2             | 0                |

---

## Important rules

- **Read-only.** This skill never modifies issues, PRs, or labels. It only reads data and writes the local report file.
- **Graceful degradation.** If any API call fails (e.g., Dependabot 403, CI unavailable), record the failure, note it in the report, and continue. Never stop the entire health check because one data source is unavailable.
- **Rate limit awareness.** If `RATE_REMAINING` drops below 50 during execution, pause and warn: "Rate limit low ({N} remaining). Recommend pausing and resuming in {estimated minutes} minutes."
- **Report always written.** Write the `.mstack/reports/{DATE}-health.md` file regardless of what the user does in Step 7. The report has value as a historical record even if no actions are taken.
- **Do not auto-invoke skills.** When the user chooses option A in Step 7, suggest the relevant skills and explain why. Do not call them automatically — the user must run them explicitly.
