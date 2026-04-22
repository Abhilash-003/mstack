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
  DETECT_DUPLICATES=$(grep "detect_duplicates:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "true")
  DETECT_BOTS=$(grep "detect_bots:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "true")
  STALE_DAYS=$(grep "stale_days:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "30")
  echo "REPO: $REPO"
  echo "DETECT_DUPLICATES: $DETECT_DUPLICATES"
  echo "DETECT_BOTS: $DETECT_BOTS"
  echo "STALE_DAYS: $STALE_DAYS"
else
  echo "REPO_CONFIGURED: false"
fi
```

If `GH_CLI` is not installed: stop and tell the user "MStack requires the GitHub CLI. Install from https://cli.github.com then re-run /triage."

If `gh auth status` shows not authenticated: stop and tell the user "gh CLI is not authenticated. Run `gh auth login` first, then re-run /triage."

If `REPO_CONFIGURED` is false: stop and tell the user "This repo is not configured for MStack yet. Run `/mstack-setup` first."

If `RATE_REMAINING` is a number less than 50: warn the user "GitHub API rate limit is low ({RATE_REMAINING} requests remaining). Triage may be throttled. Continuing anyway — watch for errors."

---

## Step 1 — Fetch open issues

```bash
gh issue list --state open \
  --json number,title,body,author,labels,createdAt,comments,assignees \
  --limit 50
```

Parse the JSON output. For each issue, check whether it already has any category labels. Category labels are those matching: `bug`, `feature`, `enhancement`, `docs`, `question`, `security`.

Issues that already have at least one category label do not need categorization triage — filter them out.

If all open issues already have category labels, print:

```
All open issues already have category labels. Nothing to triage.
```

...and stop.

If more than 20 issues need triage, note that this is a large batch. Process the newest 20 first (sort by `createdAt` descending). After completing those, ask the user:

> Triaged 20 issues. There are {N} more open issues without category labels. Continue triaging the next batch?
>
> A) Yes — triage next 20
> B) No — stop here

Store the filtered list of issues needing triage. Record the count as `ISSUES_TO_TRIAGE`.

---

## Step 2 — Fetch existing issues for duplicate comparison

Only run this step if `DETECT_DUPLICATES` is `true`.

```bash
gh issue list --state all --limit 100 \
  --json number,title,body,labels,state
```

Store this as the reference list for duplicate detection in Step 3. Call it `ALL_ISSUES`.

---

## Step 3 — Analyze each issue

For each issue in the triage list (from Step 1), perform the following analysis. Process issues one at a time.

### 3a. Fetch full issue detail

```bash
gh issue view <NUMBER> \
  --json number,title,body,author,labels,createdAt,comments,assignees,milestone,url
```

### 3b. Categorize

Assign one primary category based on the issue title and body:

| Category | When to assign |
|----------|---------------|
| `bug` | Reports unexpected behavior, errors, crashes, or regressions |
| `feature` | Requests new functionality that does not currently exist |
| `enhancement` | Requests improvement to existing functionality |
| `docs` | Documentation gaps, errors, or improvement requests |
| `question` | Asks how something works, seeks guidance or clarification |
| `security` | Reports a vulnerability, authentication issue, data leak, or security concern |

When uncertain between categories, pick the one that best matches the author's primary intent.

### 3c. Assess priority

Assign one priority level:

| Priority | Criteria |
|----------|---------|
| `critical` | Security vulnerability, data loss, crash affecting all users, or blocker for core functionality |
| `high` | Significant bug affecting many users, or a highly requested feature with a clear use case |
| `medium` | Moderate impact bug or reasonable feature request with a clear description |
| `low` | Cosmetic issues, minor improvements, nice-to-haves, or vague requests needing clarification |

### 3d. Check for duplicates

Only if `DETECT_DUPLICATES` is `true`: compare this issue's title and body against `ALL_ISSUES` from Step 2. Look for semantic similarity — not just exact matches. Consider an issue a likely duplicate if:

- The titles describe the same problem or request in different words, AND
- The bodies describe similar symptoms, environments, or use cases

If a likely duplicate is found, record the issue number(s) of the original(s).

### 3e. Bot and spam check

Only if `DETECT_BOTS` is `true`:

Read `_shared/detection.md` (relative to the MStack skill root) to load the detection heuristics.

Then check the issue author's account:

```bash
gh api "users/<AUTHOR_LOGIN>" --jq '{type: .type, public_repos: .public_repos, followers: .followers, created_at: .created_at, bio: .bio}'
```

Apply the account signals from `detection.md`. Also evaluate the issue body against the content signals. Assign a bot/spam suspicion level:

- `none` — no signals detected
- `low` — one weak signal (e.g., new account alone)
- `medium` — two or more weak signals, or one stronger signal
- `high` — explicit bot account type, or multiple strong signals

**Conservative principle:** when in doubt, use a lower suspicion level. A false positive (flagging a real user) is worse than a false negative (missing a bot). Only recommend action at `high` suspicion.

### 3f. Completeness check

Evaluate whether the issue provides enough information to act on:

- For `bug` issues: Are reproduction steps present? Is the version or environment mentioned?
- For `feature`/`enhancement`: Is there a clear problem statement or use case?
- For `question`: Is the question specific enough to answer?
- For `docs`: Is the unclear or missing section identified?
- For `security`: Is there enough detail to assess severity? (Do not request public details for security issues — flag for private handling instead.)

Mark completeness as `complete`, `needs-info`, or `incomplete`.

### 3g. Write analysis to log

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
mkdir -p "$PROJECT_ROOT/.mstack/logs"
DATE=$(date +%Y-%m-%d)
LOG_FILE="$PROJECT_ROOT/.mstack/logs/triage-$DATE.jsonl"

cat >> "$LOG_FILE" <<'JSONEOF'
{"number": <NUMBER>, "title": "<TITLE>", "category": "<CATEGORY>", "priority": "<PRIORITY>", "duplicate_of": <null or [array of numbers]>, "bot_suspicion": "<none|low|medium|high>", "completeness": "<complete|needs-info|incomplete>", "recommended_action": "<ACTION>", "timestamp": "<ISO8601>"}
JSONEOF
echo "Logged issue #<NUMBER> to $LOG_FILE"
```

Substitute actual values for all placeholders. `recommended_action` should be one of:
- `label` — apply category and priority labels
- `label+comment` — apply labels and post a request for more information
- `duplicate` — mark as duplicate and close
- `bot-flag` — flag for maintainer review as suspected bot/spam
- `security` — flag for private security handling, do not label publicly

---

## Step 4 — Present triage report

After analyzing all issues, display a triage report in table format:

```
Triage Report — {DATE}
Repo: {REPO}
Issues analyzed: {N}
=====================================================================
 #     | Title (truncated to 45 chars)         | Cat      | Pri    | Action
-------|---------------------------------------|----------|--------|------------------
 #123  | Cannot login after password reset     | bug      | high   | label+comment
 #118  | Add dark mode support                 | feature  | low    | label
 #115  | How do I configure custom domains?    | question | medium | label+comment
 #112  | Crash on startup when config missing  | bug      | crit   | label
 #109  | Update installation docs for v2       | docs     | medium | label
 #104  | Same crash as #112                    | bug      | high   | duplicate (#112)
 #098  | (suspected bot)                       | —        | —      | bot-flag
=====================================================================
Recommended actions:
  label:         {N} issues
  label+comment: {N} issues
  duplicate:     {N} issues
  bot-flag:      {N} issues
  security:      {N} issues
```

Notes on the report:
- Truncate titles to 45 characters, append `...` if longer
- For duplicates, show the original issue number in the Action column
- For bot-flagged issues, replace the title with `(suspected bot)` to avoid amplifying spam content
- For security issues, show `(security — handle privately)` in the Action column

---

## Step 5 — Approve actions

Ask the user:

> How would you like to approve these triage actions?
>
> A) Review each issue individually (confirm one by one)
> B) Batch approve all recommended actions
> C) Approve all EXCEPT bot-flagged and security issues (review those individually)
> D) Skip — take no actions now (log was already written)

### Option A — Individual review

For each issue, present:

```
Issue #<NUMBER>: <TITLE>
Category:    <CATEGORY>
Priority:    <PRIORITY>
Action:      <RECOMMENDED_ACTION>
<If duplicate>  Duplicate of: #<ORIGINAL>
<If bot>        Bot signals: <SUMMARY OF SIGNALS>
<If needs-info> Missing: <WHAT IS MISSING>

Approve? (y/n/skip/view)
  y     — execute recommended action
  n     — skip this issue, take no action
  skip  — skip remaining and stop
  view  — open issue in browser, then re-prompt
```

If user types `view`:

```bash
gh issue view <NUMBER> --web
```

Then re-display the approval prompt for the same issue.

### Option B — Batch approve

Confirm once:

> About to apply labels/comments/closes to {N} issues. Proceed? (yes/no)

If yes, execute all actions. If no, return to the approval menu.

### Option C — Approve except flagged

Execute all `label` and `label+comment` actions without further prompting. Then present each `bot-flag` and `security` issue individually using the Option A format.

### Option D — Skip

Print:

```
No actions taken. Triage log written to .mstack/logs/triage-{DATE}.jsonl
```

Stop.

---

### Executing approved actions

For each approved action, run the appropriate `gh` commands:

**Apply labels** (`label` action):

```bash
gh issue edit <NUMBER> --add-label "<CATEGORY>"
gh issue edit <NUMBER> --add-label "priority: <PRIORITY>"
# Remove needs-triage label if present
gh issue edit <NUMBER> --remove-label "needs-triage" 2>/dev/null || true
echo "Labeled #<NUMBER>: <CATEGORY>, priority: <PRIORITY>"
```

**Apply labels + request info** (`label+comment` action):

```bash
gh issue edit <NUMBER> --add-label "<CATEGORY>"
gh issue edit <NUMBER> --add-label "priority: <PRIORITY>"
gh issue edit <NUMBER> --add-label "needs-info"
gh issue edit <NUMBER> --remove-label "needs-triage" 2>/dev/null || true
gh issue comment <NUMBER> --body "<COMMENT_BODY>"
echo "Labeled and commented on #<NUMBER>"
```

The comment body should be friendly, specific about what is missing, and follow the tone guidelines from `_shared/response-guidelines.md` if it exists. Example for a bug missing reproduction steps:

> Thanks for filing this! To help us investigate, could you provide:
>
> - Steps to reproduce the issue
> - Your version of {PROJECT} and {PLATFORM/RUNTIME}
> - Any relevant error messages or logs
>
> Once we have those details, we can look into this further.

**Mark duplicate** (`duplicate` action):

```bash
gh issue edit <NUMBER> --add-label "duplicate"
gh issue comment <NUMBER> --body "This appears to be a duplicate of #<ORIGINAL>. Closing in favor of the original — please add your comments or +1 there if this is relevant to you."
gh issue close <NUMBER>
echo "Closed #<NUMBER> as duplicate of #<ORIGINAL>"
```

**Bot flag** (`bot-flag` action):

```bash
gh issue edit <NUMBER> --add-label "needs-triage"
echo "Flagged #<NUMBER> for maintainer review (suspected bot/spam)"
```

Do NOT close or comment on suspected bot issues automatically. Only label for maintainer review.

**Security flag** (`security` action):

```bash
echo "Issue #<NUMBER> flagged as potential security issue."
echo "Do NOT add public labels. Handle via private security advisory."
echo "Consider: gh api repos/{REPO}/security-advisories --method POST"
```

Do not apply any public labels to security issues. Do not post public comments. Only log the flag internally and instruct the maintainer.

---

## Step 6 — Summary

After completing all approved actions, print a final summary:

```
Triage complete — {DATE}
==================================
Issues analyzed:  {N}
Actions taken:
  Labeled:          {N}
  Labeled+commented:{N}
  Closed (dup):     {N}
  Flagged (bot):    {N}
  Flagged (sec):    {N}
  Skipped:          {N}

Log file: .mstack/logs/triage-{DATE}.jsonl
==================================
```

If any `gh` command failed during execution, list the failed issue numbers and the error so the user can retry manually.

---

## Important rules

- **Never auto-close without approval.** Every close action requires explicit user confirmation.
- **Never post comments without approval.** Even `needs-info` comments must be approved.
- **Conservative bot detection.** When uncertain about bot status, use a lower suspicion level. Flag only at `high` confidence. Closing or commenting on a real user's issue is worse than letting a bot through.
- **Security issues are private.** Never add public labels, post public comments, or expose details. Always route to private advisory handling.
- **Rate limit awareness.** If `RATE_REMAINING` drops below 50 during execution, pause and warn: "Rate limit low ({N} remaining). Recommend pausing and resuming in {estimated minutes} minutes."
- **Large repos.** If more than 20 issues need triage, process the newest 20 first and ask before continuing.
- **Log always.** Write the `.jsonl` log regardless of whether the user approves actions. The analysis record has value even if no labels are applied.
