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
  REQUIRE_TESTS=$(grep "require_tests:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "true")
  SECURITY_SCAN=$(grep "security_scan:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "true")
  MAX_FILES_WARN=$(grep "max_files_warn:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "50")
  echo "REPO: $REPO"
  echo "REQUIRE_TESTS: $REQUIRE_TESTS"
  echo "SECURITY_SCAN: $SECURITY_SCAN"
  echo "MAX_FILES_WARN: $MAX_FILES_WARN"
else
  echo "REPO_CONFIGURED: false"
fi
```

If `GH_CLI` is not installed: stop and tell the user "MStack requires the GitHub CLI. Install from https://cli.github.com then re-run /review-prs."

If `gh auth status` shows not authenticated: stop and tell the user "gh CLI is not authenticated. Run `gh auth login` first, then re-run /review-prs."

If `REPO_CONFIGURED` is false: stop and tell the user "This repo is not configured for MStack yet. Run `/mstack-setup` first."

If `RATE_REMAINING` is a number less than 50: warn the user "GitHub API rate limit is low ({RATE_REMAINING} requests remaining). PR review may be throttled. Continuing anyway — watch for errors."

---

## Step 1 — Fetch open PRs

```bash
gh pr list --state open \
  --json number,title,author,isDraft,reviewDecision,createdAt,headRefName,changedFiles,additions,deletions \
  --limit 50
```

Parse the JSON output. Apply these filters:

- **Exclude drafts**: skip any PR where `isDraft` is `true`.
- **Exclude already-approved**: skip any PR where `reviewDecision` is `"APPROVED"`.

If the user invoked this skill with a specific PR number (e.g., `/review-prs 42`), process only that PR. Skip the filters above for a directly specified PR number.

Store the filtered list as `PRS_TO_REVIEW`. Record the count as `PR_COUNT`.

If `PR_COUNT` is 0, print:

```
No open PRs need pre-screening (drafts and approved PRs excluded).
```

...and stop.

If `PR_COUNT` is greater than 20, note it is a large batch. Process the 20 most recently created PRs first (sort by `createdAt` descending). After completing those, ask the user:

> Reviewed 20 PRs. There are {N} more open PRs. Continue reviewing the next batch?
>
> A) Yes — review next 20
> B) No — stop here

---

## Step 2 — Analyze each PR

For each PR in `PRS_TO_REVIEW`, perform the following sub-steps. Process PRs one at a time.

### 2a. Read diff

```bash
gh pr diff <NUMBER>
```

Count total lines changed (additions + deletions). Store as `DIFF_LINES`.

If `DIFF_LINES` is greater than 1000: warn "Large diff ({DIFF_LINES} lines). Focusing analysis on critical files first." Prioritize files matching these patterns (in order): source/library files, configuration files, dependency manifests, security-relevant paths. Defer style-only files.

If the number of changed files exceeds `MAX_FILES_WARN`: note "This PR touches {N} files — consider asking the author to split it."

### 2b. Security scan

Only run this sub-step if `SECURITY_SCAN` is `true`.

Scan the diff for:

**Hardcoded secrets** — look for patterns matching:
- `api_key\s*=\s*["'][^"']{8,}["']`
- `token\s*=\s*["'][^"']{8,}["']`
- `secret\s*=\s*["'][^"']{8,}["']`
- `password\s*=\s*["'][^"']{8,}["']`
- `private_key`, `BEGIN RSA`, `BEGIN EC`, `BEGIN OPENSSH`
- AWS/GCP/Azure key prefixes (`AKIA`, `AIza`, etc.)

**Injection risks**:
- SQL injection: unparameterized string concatenation into queries (`"SELECT" + variable`, f-strings in SQL calls)
- Command injection: `exec(`, `eval(`, `subprocess.call(...shell=True`, `os.system(`, backtick execution

**New dependencies**: changes to `package.json`, `requirements.txt`, `Gemfile`, `go.mod`, `pom.xml`, `Cargo.toml`, or similar. Note any new packages added.

**Permission changes**: changes to file modes, `chmod`, CORS config, authentication middleware, or authorization checks.

For each finding, classify as:
- `critical` — hardcoded secret or explicit credential
- `high` — injection risk, unsafe permission escalation
- `medium` — suspicious pattern that may be a false positive
- `low` — new dependency added (informational)

Security findings always set the PR risk level to at least HIGH. Critical findings set it to CRITICAL.

### 2c. Test coverage

Only run this sub-step if `REQUIRE_TESTS` is `true`.

From the diff, identify:
- **Source files changed**: files in `src/`, `lib/`, `app/`, or similar, with language extensions (`.py`, `.js`, `.ts`, `.rb`, `.go`, `.java`, `.rs`, etc.)
- **Test files changed**: files in `test/`, `tests/`, `spec/`, `__tests__/`, matching `*.test.*`, `*_test.*`, `*_spec.*`, or similar

Compare counts:
- If source files were changed but **no** test files were changed: flag as "no tests added"
- If source files added significantly outnumber test files: flag as "test coverage gap"
- If test files exist and appear proportional: flag as "tests present"

Note: documentation-only PRs, config-only PRs, and PRs touching only non-code files are exempt from this check.

### 2d. Breaking change detection

Scan the diff for:

- **Removed functions or methods**: lines starting with `-` that define a function (`def `, `function `, `func `, `public `, `export function`, etc.)
- **Renamed exports**: a function/class removed and a new one added with a different name in the same file
- **Changed function signatures**: parameter lists altered on existing public functions
- **Removed API endpoints**: routes removed (`app.get(`, `@app.route`, `router.delete(`, etc.)
- **Removed or renamed config keys**: keys removed from config schemas, environment variable names changed
- **Major version bumps** in dependency manifests

For each detected breaking change, note the file and the change. Flag the PR with "potential breaking changes" if any are found.

### 2e. Code quality

Review the diff for:

- **Large PR**: if `DIFF_LINES` > 500 and not already flagged for >1000, note it as a large change that may be hard to review thoroughly
- **Obvious bugs**: null pointer dereferences on values that may be nil/null/undefined, off-by-one errors in loops, unclosed resources (file handles, DB connections), unhandled errors/exceptions
- **Dead code**: functions or variables added in this PR that are never called/referenced
- **Style concerns**: inconsistent naming conventions, deeply nested logic (>4 levels), very long functions (>100 lines added in one block)

Only flag quality issues that are clear from the diff — do not speculate about issues in unchanged code.

### 2f. Bot detection

Read `_shared/detection.md` (relative to the MStack skill root) to load detection heuristics.

Check the PR author's account:

```bash
gh api "users/<AUTHOR_LOGIN>" --jq '{type: .type, public_repos: .public_repos, followers: .followers, created_at: .created_at, bio: .bio}'
```

Apply the account signals and PR-specific signals from `detection.md`. Assign a bot/spam suspicion level:

- `none` — no signals detected
- `low` — one weak signal (e.g., new account alone, or `patch-1` branch name alone)
- `medium` — two or more weak signals, or one stronger signal
- `high` — explicit bot account type, or multiple strong signals (generated code + no tests + trivial change + new account)

**Conservative principle:** false positives (flagging a real contributor) are worse than false negatives. Only elevate to `high` with clear evidence. Be fair to new contributors — a first-time contributor with a new account submitting a legitimate fix is not a bot.

### 2g. CI status

```bash
gh pr checks <NUMBER> 2>/dev/null || echo "CI_STATUS: unavailable"
```

Parse the output. Summarize:
- All checks passing: `ci: passing`
- One or more failing: `ci: failing ({list of failing check names})`
- Checks still running: `ci: pending`
- No CI configured or unavailable: `ci: unknown`

---

## Step 3 — Generate review summary

After completing all sub-steps for a PR, assign an overall **risk level**:

| Risk | Criteria |
|------|---------|
| `critical` | Hardcoded secrets, explicit credentials, or malicious-looking code |
| `high` | Injection vulnerabilities, unsafe permission changes, security findings of any kind, or CI failing on security-related checks |
| `medium` | Breaking changes detected, no tests added for non-trivial source changes, multiple quality issues, bot suspicion `medium` or above |
| `low` | Clean diff, tests present or exempt, no security findings, no breaking changes, CI passing |

Then write the log entry:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
mkdir -p "$PROJECT_ROOT/.mstack/logs"
DATE=$(date +%Y-%m-%d)
LOG_FILE="$PROJECT_ROOT/.mstack/logs/review-$DATE.jsonl"

cat >> "$LOG_FILE" <<'JSONEOF'
{"number": <NUMBER>, "title": "<TITLE>", "author": "<AUTHOR>", "risk": "<low|medium|high|critical>", "security_findings": <null or [array of findings]>, "test_coverage": "<present|gap|none|exempt>", "breaking_changes": <true|false>, "bot_suspicion": "<none|low|medium|high>", "ci_status": "<passing|failing|pending|unknown>", "diff_lines": <NUMBER>, "recommended_action": "<ACTION>", "timestamp": "<ISO8601>"}
JSONEOF
echo "Logged PR #<NUMBER> to $LOG_FILE"
```

`recommended_action` should be one of:
- `approve-ready` — low risk, looks good, no blockers
- `request-changes` — medium or high risk, specific issues to address
- `security-review` — critical or high security findings, needs security-focused review
- `needs-tests` — otherwise fine but missing test coverage
- `bot-flag` — high bot suspicion, flag for maintainer review
- `skip` — already has sufficient reviews or is out of scope

---

## Step 4 — Present review report

After analyzing all PRs, display a report in table format:

```
PR Review Report — {DATE}
Repo: {REPO}
PRs analyzed: {N}
=============================================================================
 #     | Risk     | Title (truncated to 40 chars)          | CI       | Recommendation
-------|----------|----------------------------------------|----------|---------------------
 #47   | CRITICAL | Add payment integration                | failing  | security-review
 #43   | HIGH     | Refactor auth middleware               | passing  | request-changes
 #39   | MEDIUM   | Update user profile endpoint           | passing  | needs-tests
 #35   | MEDIUM   | Add bulk import feature                | pending  | request-changes
 #28   | LOW      | Fix typo in README                     | passing  | approve-ready
 #21   | LOW      | Update dependencies                    | passing  | approve-ready
=============================================================================
Risk breakdown:
  critical: {N}   high: {N}   medium: {N}   low: {N}

Security findings: {N} PRs have security concerns — review these first.
```

Notes on the report:
- Truncate titles to 40 characters, append `...` if longer
- Show risk in uppercase for emphasis
- List CRITICAL and HIGH risk PRs first, then MEDIUM, then LOW
- If no security findings, omit the security findings line at the bottom

---

## Step 5 — Approve review actions

Ask the user:

> How would you like to handle review comments?
>
> A) Review each PR individually (confirm comment before posting)
> B) Review HIGH and CRITICAL risk PRs only
> C) Post all draft comments now (after one final confirmation)
> D) Skip — log was written, no comments posted

### Option A — Individual review

For each PR, present:

```
PR #<NUMBER>: <TITLE>
Author:    <AUTHOR>
Risk:      <RISK LEVEL>
CI:        <CI STATUS>
Diff:      <DIFF_LINES> lines changed

Analysis:
  Security:        <findings summary or "none">
  Tests:           <present / gap / none / exempt>
  Breaking changes:<yes (details) / no>
  Bot suspicion:   <none / low / medium / high>
  Quality notes:   <list or "none">

Draft comment:
---------
<DRAFT COMMENT BODY>
---------

Action? (approve/edit/skip/view)
  approve — post this comment as-is
  edit    — let me revise the comment before posting
  skip    — skip this PR, post no comment
  view    — open PR in browser, then re-prompt
```

If user chooses `view`:

```bash
gh pr view <NUMBER> --web
```

Then re-display the prompt for the same PR.

If user chooses `edit`: show the draft comment and allow free-form editing before posting.

Post approved comments via:

```bash
gh pr review <NUMBER> --comment --body "<COMMENT_BODY>"
echo "Comment posted on PR #<NUMBER>"
```

**NEVER use `gh pr review --approve` or `gh pr merge`. Never auto-approve or auto-merge any PR.**

### Option B — High/critical only

Filter `PRS_TO_REVIEW` to those with risk level `high` or `critical`. Present each using the Option A format.

### Option C — Post all (after approval)

Show a summary of all draft comments. Ask once:

> About to post review comments on {N} PRs. Proceed? (yes/no)

If yes, post all comments. If no, return to the approval menu.

### Option D — Skip

Print:

```
No comments posted. Review log written to .mstack/logs/review-{DATE}.jsonl
```

Stop.

---

## Step 6 — Summary

After completing all approved actions, print a final summary:

```
PR review complete — {DATE}
==================================
PRs analyzed:    {N}
Comments posted: {N}
Skipped:         {N}

Risk breakdown:
  Critical: {N}
  High:     {N}
  Medium:   {N}
  Low:      {N}

Log file: .mstack/logs/review-{DATE}.jsonl
==================================
```

If any `gh` command failed during execution, list the failed PR numbers and the error so the user can retry manually.

---

## Important rules

- **Never approve or merge PRs.** Only post informational review comments. Never use `--approve`, `--request-changes` without user confirmation, or `gh pr merge`.
- **Security findings always raise risk.** Any security finding sets the PR risk to HIGH at minimum. Critical findings (hardcoded secrets) always set risk to CRITICAL.
- **Be fair to new contributors.** A new account is not evidence of bad intent. Apply the conservative principle from `_shared/detection.md`. A first-time contributor with a small, clean PR should receive a welcoming review.
- **Large diffs (>1000 lines): warn and focus.** Do not attempt a full review of enormous diffs. Focus on security-relevant, API-boundary, and high-risk files. Note explicitly that a thorough review of large PRs may require a human reading the full diff.
- **Log always.** Write the `.jsonl` log regardless of whether the user approves posting any comments. The analysis record is valuable even with no actions taken.
- **Rate limit awareness.** If `RATE_REMAINING` drops below 50 during execution, pause and warn: "Rate limit low ({N} remaining). Recommend pausing and resuming in {estimated minutes} minutes."
- **Draft comments, not final verdicts.** All generated review text is a draft for the human maintainer to approve, edit, or discard. Frame the tone as helpful and constructive, following `_shared/response-guidelines.md` if it exists.
