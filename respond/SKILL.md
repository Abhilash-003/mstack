---
name: respond
version: 0.1.0
description: |
  Draft tone-matched maintainer responses for open issues and pull requests.
  Uses project response templates, learns the maintainer's voice from recent
  comments, and requires human approval before posting anything.
  Use when asked to "respond to issues", "draft replies", "handle issue responses",
  "write PR responses", or "respond to contributors".
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
  echo "REPO: $REPO"
else
  echo "REPO_CONFIGURED: false"
fi
```

If `GH_CLI` is not installed: stop and tell the user "MStack requires the GitHub CLI. Install from https://cli.github.com then re-run /respond."

If `gh auth status` shows not authenticated: stop and tell the user "gh CLI is not authenticated. Run `gh auth login` first, then re-run /respond."

If `REPO_CONFIGURED` is false: stop and tell the user "This repo is not configured for MStack yet. Run `/mstack-setup` first."

If `RATE_REMAINING` is a number less than 50: warn the user "GitHub API rate limit is low ({RATE_REMAINING} requests remaining). Response drafting may be throttled. Continuing anyway — watch for errors."

---

## Step 1 — Identify what needs responses

If the user invoked this skill with specific issue or PR numbers (e.g., `/respond 42 #17`), use those items only. Skip the discovery queries below and go directly to Step 2.

Otherwise, find items without a maintainer response:

```bash
# Issues with no comments at all (likely no maintainer response)
gh issue list --state open \
  --json number,title,author,createdAt,comments,labels \
  --limit 50

# PRs with no review yet
gh pr list --state open \
  --json number,title,author,createdAt,reviewDecision,isDraft,reviews \
  --limit 50
```

From the issue list, identify issues where `comments` is an empty array — these have received no response at all. Also include issues where a maintainer comment has not been made (if comment authors can be compared against known maintainers from config).

From the PR list, identify PRs where `reviewDecision` is `null` or `""` and `isDraft` is `false` — these are open, non-draft PRs that have not been reviewed.

If the user invoked with `--issues-only` or `--prs-only`, filter accordingly.

Store the combined list as `ITEMS_TO_RESPOND`. Record the count as `ITEM_COUNT`.

If `ITEM_COUNT` is 0, print:

```
No open issues or PRs need a response right now.
```

...and stop.

If `ITEM_COUNT` is greater than 10, note it is a large batch. Process the 10 oldest items first (sort by `createdAt` ascending — oldest first, as they have been waiting longest). After completing those, ask the user:

> Drafted responses for 10 items. There are {N} more waiting. Continue with the next batch?
>
> A) Yes — draft next 10
> B) No — stop here

---

## Step 2 — Learn project tone

Read recent maintainer activity to calibrate tone before drafting anything.

```bash
# Fetch the 10 most recent issue comments from humans (not bots)
gh api "repos/{REPO}/issues/comments?sort=created&direction=desc&per_page=10"
```

Replace `{REPO}` with the repo value from config.

From these comments, note:
- **Formality**: formal prose vs. casual/conversational
- **Emoji use**: none, occasional, or frequent
- **Length**: short (1–3 sentences), medium (a paragraph), or long (multiple paragraphs)
- **Greeting style**: do responses open with "Thanks", "Hey", "Hi {name}", or no greeting at all?
- **Punctuation habits**: Oxford comma, em-dashes, sentence-ending periods on short responses
- **Technical formatting**: inline `code`, fenced blocks, bold headers, or plain prose

Store these observations as `TONE_PROFILE`. You will apply this profile when drafting each response.

Then read the project's response guidelines:

Read `_shared/response-guidelines.md` (relative to the MStack skill root). If it does not exist, proceed with the core principles: respectful, specific, actionable, brief, grateful.

---

## Step 3 — Draft responses

For each item in `ITEMS_TO_RESPOND`, perform the following. Process items one at a time.

### 3a. Fetch full content

For an issue:

```bash
gh issue view <NUMBER> \
  --json number,title,body,author,labels,createdAt,comments,assignees,url
```

For a PR:

```bash
gh pr view <NUMBER> \
  --json number,title,body,author,createdAt,labels,reviews,reviewDecision,isDraft,url,headRefName,changedFiles,additions,deletions
```

### 3b. Identify the scenario

Read the issue or PR body and any existing comments. Classify it into one of these scenarios:

| Scenario | When it applies |
|----------|----------------|
| `needs-info` | Report lacks reproduction steps, version info, or a clear description of the problem |
| `duplicate` | Same problem or request already tracked in another open issue |
| `welcome` | A first-time contributor's pull request (check: no prior PRs merged or authored in this repo) |
| `bug-confirmed` | A bug report where the problem is clear and reproducible from the description |
| `wont-fix` | A valid issue or feature request that falls outside project scope or priorities |
| `stale-close` | An issue that has been open a long time with no activity and is no longer relevant |
| `pr-feedback` | A PR that needs review feedback — changes requested, questions, or approval notes |
| `general-thanks` | A simple thank-you or acknowledgment (e.g., closing a completed issue, thanking a merged PR) |

If the scenario is ambiguous, choose the one that best describes what the contributor most needs to hear next.

### 3c. Read the matching template

For the identified scenario, read the corresponding template file:

- `needs-info` → read `templates/responses/needs-info.md`
- `duplicate` → read `templates/responses/duplicate.md`
- `welcome` → read `templates/responses/welcome.md`
- `bug-confirmed` → read `templates/responses/bug-confirmed.md`
- `wont-fix` → read `templates/responses/wont-fix.md`
- `stale-close` → read `templates/responses/stale-close.md`
- `pr-feedback` → no fixed template; draft based on tone profile and response guidelines
- `general-thanks` → no fixed template; draft a brief, warm closing comment

All template paths are relative to the MStack skill root.

### 3d. Draft the response

Using the template structure as a base, write a response that:

1. **Is specific to this item** — references the actual issue title, the specific missing information, the exact PR changes, or the real reason for declining. Never use placeholder text in a draft. Fill every variable.
2. **Matches the `TONE_PROFILE`** — if the project is casual with emoji, reflect that. If it is formal and terse, match that.
3. **Is actionable** — the contributor should know exactly what to do next (or what will happen next).
4. **Respects the contributor** — no condescension, no blame, no corporate language. Follow the "Never Do" list from the response guidelines.
5. **Does not mention AI** — the response will be posted under the maintainer's name. Write it as if the maintainer wrote it.

For `needs-info` responses, identify the specific missing items from the issue body. Do not list things that are already present.

For `duplicate` responses, look up the original issue number and title:

```bash
gh issue list --state all --json number,title,labels --limit 100
```

Find the best match. Include the number and title in the draft.

For `welcome` responses on PRs, check whether the author has contributed before:

```bash
gh pr list --state all --author "<AUTHOR_LOGIN>" --json number,mergedAt --limit 5
```

If they have zero prior PRs, this is genuinely a first-time contributor. Use the welcome template. If they have prior contributions, use `pr-feedback` instead.

For `stale-close` responses, calculate how many days the issue has been open:

```bash
CREATED=$(gh issue view <NUMBER> --json createdAt --jq '.createdAt')
DAYS_OPEN=$(( ( $(date +%s) - $(date -d "$CREATED" +%s 2>/dev/null || date -j -f "%Y-%m-%dT%H:%M:%SZ" "$CREATED" +%s) ) / 86400 ))
echo "DAYS_OPEN: $DAYS_OPEN"
```

Use the actual number in the draft.

Store the completed draft as `DRAFT_<NUMBER>`.

---

## Step 4 — Present drafts for approval

For each drafted response, present it to the maintainer for approval using AskUserQuestion.

Show the following for each item:

```
-----------------------------------------------------------------------
Issue #<NUMBER>: <TITLE>    [or PR #<NUMBER>: <TITLE>]
Author:   <AUTHOR_LOGIN>
Scenario: <SCENARIO>
URL:      <URL>

Draft response:
-----------------------------------------------------------------------
<DRAFT RESPONSE TEXT>
-----------------------------------------------------------------------

What would you like to do?
  A) Post this response as-is
  B) Edit before posting
  C) Skip — post nothing for this item
```

If the maintainer chooses **A — Post**:

For an issue:
```bash
gh issue comment <NUMBER> --body "<RESPONSE_BODY>"
echo "Comment posted on issue #<NUMBER>"
```

For a PR:
```bash
gh pr review <NUMBER> --comment --body "<RESPONSE_BODY>"
echo "Comment posted on PR #<NUMBER>"
```

Record result as `posted`.

If the maintainer chooses **B — Edit**:

Ask: "What would you like to change?" Accept free-form input. Incorporate the requested changes into the draft. Show the revised draft and ask again:

> Here is the revised draft:
>
> ---
> {REVISED_DRAFT}
> ---
>
> Post this version? (yes / edit again / skip)

If yes, post using the same `gh` command above. Record result as `edited+posted`.

If they choose to edit again, repeat the revision loop.

If they choose to skip at any point, record result as `skipped`.

If the maintainer chooses **C — Skip**:

Record result as `skipped`. Move to the next item.

**Never post a response without explicit maintainer approval.**

---

## Step 5 — Summary

After processing all items, print a final summary:

```
Response session complete — {DATE}
==================================
Items reviewed:  {TOTAL}
  Posted:        {N}
  Edited+posted: {N}
  Skipped:       {N}
==================================
```

If any `gh` command failed during posting, list the failed item numbers and errors so the maintainer can retry manually.

---

## Important rules

- **Never post without approval.** Every response — including edits — requires explicit maintainer confirmation before it is posted.
- **Never be dismissive.** Even when closing stale issues, declining features, or marking duplicates, acknowledge the contributor's effort. A one-line dismissal leaves contributors feeling ignored.
- **No AI disclaimers.** Responses are drafted for the maintainer to post under their own name. Never include phrases like "As an AI" or "I should note that I generated this." Write as the maintainer.
- **Be specific, not generic.** Drafts must reference the actual issue title, specific missing details, real PR changes, or concrete reasons. A generic template filled with placeholders is not a draft — it is a skeleton.
- **Match the project's voice.** Mismatched tone (unexpectedly formal in a casual project, or unexpectedly casual in a formal one) signals something is off. Spend time on Step 2 and apply the tone profile consistently.
- **Rate limit awareness.** If `RATE_REMAINING` drops below 50 during execution, pause and warn: "Rate limit low ({N} remaining). Recommend pausing and resuming in {estimated minutes} minutes."
- **One item at a time.** Present each draft individually. Do not batch-present multiple drafts in a single prompt — this encourages rubber-stamping without reading.
