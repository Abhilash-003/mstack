---
name: maintain
version: 0.1.0
description: |
  Full maintenance pipeline orchestrator: triage → PR review → response drafting → health check.
  Chains all MStack skills with human checkpoints between each phase.
  Use when asked to "run maintenance", "morning maintenance", "full pipeline",
  "process everything", or "run all skills".
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
  echo "REPO: $REPO"
else
  echo "REPO_CONFIGURED: false"
fi

# Quick stats
if command -v gh >/dev/null 2>&1 && [ -f "$PROJECT_ROOT/.mstack/config.yml" ]; then
  OPEN_ISSUES=$(gh issue list --state open --json number --jq 'length' 2>/dev/null || echo "?")
  OPEN_PRS=$(gh pr list --state open --json number --jq 'length' 2>/dev/null || echo "?")
  echo "OPEN_ISSUES: $OPEN_ISSUES"
  echo "OPEN_PRS: $OPEN_PRS"
fi
```

If `GH_CLI` is not installed: stop and tell the user "MStack requires the GitHub CLI. Install from https://cli.github.com then re-run /maintain."

If `gh auth status` shows not authenticated: stop and tell the user "gh CLI is not authenticated. Run `gh auth login` first, then re-run /maintain."

If `REPO_CONFIGURED` is false: stop and tell the user "This repo is not configured for MStack yet. Run `/mstack-setup` first."

If `RATE_REMAINING` is a number less than 100: warn the user "GitHub API rate limit is low ({RATE_REMAINING} requests remaining). Running the full pipeline is rate-intensive. Proceeding — watch for throttle warnings during each phase."

Tell the user:

> Starting full maintenance session for {REPO}.
> Open issues: ~{OPEN_ISSUES} | Open PRs: ~{OPEN_PRS}
>
> I'll walk you through: triage → PR review → responses → health check.
> You have a checkpoint between each phase to continue, skip ahead, or stop.

---

## State tracking

Before starting, initialize the session state file:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
DATE=$(date +%Y-%m-%d)
SESSION_FILE="$PROJECT_ROOT/.mstack/logs/maintain-$DATE.json"
mkdir -p "$PROJECT_ROOT/.mstack/logs"

cat > "$SESSION_FILE" <<'STATEEOF'
{
  "date": "<DATE>",
  "repo": "<REPO>",
  "phases": {
    "triage":  {"status": "pending", "issues_processed": 0, "labeled": 0, "closed": 0},
    "review":  {"status": "pending", "prs_reviewed": 0, "commented": 0},
    "respond": {"status": "pending", "drafted": 0, "posted": 0},
    "health":  {"status": "pending", "report_path": null}
  }
}
STATEEOF
echo "Session state initialized: $SESSION_FILE"
```

Substitute actual values for `<DATE>` and `<REPO>`. Update this file at the end of each phase.

---

## Phase 1: Issue Triage

Find the MStack skills directory. The skill root is the parent directory of this file's location. Read the triage skill:

```bash
SKILL_ROOT="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
echo "SKILL_ROOT: $SKILL_ROOT"
TRIAGE_SKILL="$SKILL_ROOT/triage/SKILL.md"
if [ -f "$TRIAGE_SKILL" ]; then
  echo "TRIAGE_SKILL: found"
else
  echo "TRIAGE_SKILL: not found at $TRIAGE_SKILL"
fi
```

If the triage skill file is not found, show the error:

> Phase 1 failed: Could not locate triage/SKILL.md at {path}.

Then use AskUserQuestion to ask:

> A) Retry — try a different path
> B) Skip Phase 1 and continue to Phase 2
> C) Stop here

If user chooses A, ask them to provide the correct MStack skill root path, then re-attempt. If user chooses B, update state file (`triage.status = "skipped"`) and jump to Phase 2. If user chooses C, jump to Final Summary.

Read `$TRIAGE_SKILL`. Follow all instructions in that file from **Step 1** onward, skipping its Preamble section (environment checks have already been run).

Track the following during execution (these values will be collected from the triage skill's Step 6 summary output):
- `TRIAGE_ISSUES_PROCESSED` — issues analyzed
- `TRIAGE_LABELED` — issues labeled (including labeled+commented)
- `TRIAGE_CLOSED` — issues closed as duplicates

After the triage skill's Step 6 summary is displayed, update the session state:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
DATE=$(date +%Y-%m-%d)
SESSION_FILE="$PROJECT_ROOT/.mstack/logs/maintain-$DATE.json"
# Update triage phase status in the JSON file
echo "Triage phase complete. Updating session state."
```

### Phase 1 checkpoint

Use AskUserQuestion to ask:

> Phase 1 complete — Issue Triage.
>
> What would you like to do next?
>
> A) Continue to Phase 2: PR Review
> B) Skip Phase 2, jump to Phase 3: Response Drafting
> C) Skip to Phase 4: Health Check
> D) Stop here — end maintenance session

If user chooses D, jump to Final Summary.

---

## Phase 2: PR Review

Find and read the review-prs skill:

```bash
SKILL_ROOT="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
REVIEW_SKILL="$SKILL_ROOT/review/SKILL.md"
if [ -f "$REVIEW_SKILL" ]; then
  echo "REVIEW_SKILL: found"
else
  echo "REVIEW_SKILL: not found at $REVIEW_SKILL"
fi
```

If the review skill file is not found, show the error:

> Phase 2 failed: Could not locate review/SKILL.md at {path}.

Then use AskUserQuestion to ask:

> A) Retry — try a different path
> B) Skip Phase 2 and continue to Phase 3
> C) Stop here

If user chooses B, update state file (`review.status = "skipped"`) and jump to Phase 3. If user chooses C, jump to Final Summary.

Read `$REVIEW_SKILL`. Follow all instructions in that file from **Step 1** onward, skipping its Preamble section.

Track the following from the review skill's Step 6 summary:
- `REVIEW_PRS_REVIEWED` — PRs analyzed
- `REVIEW_COMMENTED` — comments posted

After the review skill's Step 6 summary is displayed, update the session state:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
DATE=$(date +%Y-%m-%d)
SESSION_FILE="$PROJECT_ROOT/.mstack/logs/maintain-$DATE.json"
echo "Review phase complete. Updating session state."
```

### Phase 2 checkpoint

Use AskUserQuestion to ask:

> Phase 2 complete — PR Review.
>
> What would you like to do next?
>
> A) Continue to Phase 3: Response Drafting
> B) Skip Phase 3, jump to Phase 4: Health Check
> C) Stop here — end maintenance session

If user chooses C, jump to Final Summary.

---

## Phase 3: Response Drafting

Find and read the respond skill:

```bash
SKILL_ROOT="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
RESPOND_SKILL="$SKILL_ROOT/respond/SKILL.md"
if [ -f "$RESPOND_SKILL" ]; then
  echo "RESPOND_SKILL: found"
else
  echo "RESPOND_SKILL: not found at $RESPOND_SKILL"
fi
```

If the respond skill file is not found, show the error:

> Phase 3 failed: Could not locate respond/SKILL.md at {path}.

Then use AskUserQuestion to ask:

> A) Retry — try a different path
> B) Skip Phase 3 and continue to Phase 4
> C) Stop here

If user chooses B, update state file (`respond.status = "skipped"`) and jump to Phase 4. If user chooses C, jump to Final Summary.

Read `$RESPOND_SKILL`. Follow all instructions in that file from **Step 1** onward, skipping its Preamble section.

Track the following from the respond skill's Step 5 summary:
- `RESPOND_DRAFTED` — total items reviewed (posted + edited+posted)
- `RESPOND_POSTED` — items where a response was posted (posted + edited+posted)

After the respond skill's Step 5 summary is displayed, update the session state:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
DATE=$(date +%Y-%m-%d)
SESSION_FILE="$PROJECT_ROOT/.mstack/logs/maintain-$DATE.json"
echo "Respond phase complete. Updating session state."
```

### Phase 3 checkpoint

Use AskUserQuestion to ask:

> Phase 3 complete — Response Drafting.
>
> What would you like to do next?
>
> A) Continue to Phase 4: Health Check
> B) Stop here — end maintenance session

If user chooses B, jump to Final Summary.

---

## Phase 4: Health Check

Find and read the health skill:

```bash
SKILL_ROOT="$(cd "$(dirname "$0")/.." 2>/dev/null && pwd || echo "$HOME/.claude/skills/mstack")"
HEALTH_SKILL="$SKILL_ROOT/health/SKILL.md"
if [ -f "$HEALTH_SKILL" ]; then
  echo "HEALTH_SKILL: found"
else
  echo "HEALTH_SKILL: not found at $HEALTH_SKILL"
fi
```

If the health skill file is not found, show the error:

> Phase 4 failed: Could not locate health/SKILL.md at {path}.

Then use AskUserQuestion to ask:

> A) Retry — try a different path
> B) Skip Phase 4 and go to Final Summary
> C) Stop here

If user chooses B or C, jump to Final Summary.

Read `$HEALTH_SKILL`. Follow all instructions in that file from **Step 1** onward, skipping its Preamble section.

Note: The health skill's Step 7 prompt asks what to do next. When running inside /maintain, after the user responds to Step 7 in the health skill, continue to Final Summary rather than stopping — the full pipeline session wraps up there.

Track the following from health skill execution:
- `HEALTH_REPORT_PATH` — path to the generated report file (`.mstack/reports/{DATE}-health.md`)

Update the session state after health completes:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
DATE=$(date +%Y-%m-%d)
SESSION_FILE="$PROJECT_ROOT/.mstack/logs/maintain-$DATE.json"
echo "Health phase complete. Updating session state."
```

---

## Final Summary

Collect the results from each phase that ran. For phases that were skipped or not reached, show "—".

Print:

```
=== Maintenance Session Complete: {REPO} ===
Date: {DATE}
Triage:    {TRIAGE_ISSUES_PROCESSED} issues processed, {TRIAGE_LABELED} labeled, {TRIAGE_CLOSED} closed
Review:    {REVIEW_PRS_REVIEWED} PRs reviewed, {REVIEW_COMMENTED} commented
Responses: {RESPOND_DRAFTED} drafted, {RESPOND_POSTED} posted
Health:    Report at .mstack/reports/{DATE}-health.md
Top follow-ups: {TOP_FOLLOW_UPS}
```

For `Top follow-ups`, list up to 3 of the most important action items across all phases:

- From triage: any security-flagged issues or bot-flagged issues still needing manual review
- From review: any CRITICAL or HIGH risk PRs that were flagged but not yet resolved
- From respond: any items that were skipped and still need a response
- From health: the top critical or warning items from the health report

If no follow-ups exist, write "None — all items processed."

Then write the final session state to disk:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
DATE=$(date +%Y-%m-%d)
SESSION_FILE="$PROJECT_ROOT/.mstack/logs/maintain-$DATE.json"
echo "Session complete. Final state written to: $SESSION_FILE"
```

---

## Important rules

- **This skill reads other skill files and follows them inline.** Do NOT re-implement triage, review, respond, or health logic here. Read each skill's SKILL.md and execute it directly.
- **Every phase boundary is a human checkpoint.** Never proceed to the next phase without explicit user confirmation via AskUserQuestion.
- **If any phase fails, show the error and offer Retry/Skip/Stop.** Never silently skip a phase or proceed past an error without the user's knowledge.
- **State is written to disk at each phase.** The `.mstack/logs/maintain-{DATE}.json` file records which phases ran and their outcomes. This is the record of the session.
- **Starts fresh each time.** There is no resume functionality. Each invocation of /maintain begins a new session. If a previous session file exists for today's date, overwrite it.
- **Do not re-run the preamble inside sub-skills.** Environment checks (gh CLI, auth, repo config, rate limit) have already been run at the /maintain level. Skip the Preamble sections of triage, review, respond, and health.
- **Rate limit is shared across phases.** Each phase makes multiple API calls. If rate limit warnings appear mid-session, surface them to the user and let them decide whether to pause or continue.
- **Track counts accurately.** The Final Summary should reflect real numbers from each phase's own summary output, not estimates. If a count is unavailable (phase skipped or errored), show "—" rather than a fabricated number.
