# gh CLI Helper Patterns

Common `gh` CLI commands used across MStack skills. All commands assume the current directory is a git repo with a configured remote on GitHub.

---

## Fetching Issues

List open issues with full JSON detail:

```bash
gh issue list --state open --limit 100 \
  --json number,title,body,labels,author,createdAt,updatedAt,comments
```

View a single issue:

```bash
gh issue view 42 --json number,title,body,labels,author,createdAt,comments
```

Extract just the author login and account creation date via the API:

```bash
gh issue view 42 --json author --jq '.author.login'
```

List issues with a specific label:

```bash
gh issue list --label "bug" --state open \
  --json number,title,author,createdAt
```

Filter issues created in the last 7 days:

```bash
gh issue list --state open \
  --json number,title,author,createdAt \
  --jq '[.[] | select(.createdAt > (now - 604800 | todate))]'
```

---

## Fetching Pull Requests

List open PRs:

```bash
gh pr list --state open --limit 100 \
  --json number,title,body,author,createdAt,labels,isDraft,headRefName,baseRefName
```

View a single PR:

```bash
gh pr view 99 --json number,title,body,author,createdAt,labels,commits,files,reviews,statusCheckRollup
```

Get the diff for a PR:

```bash
gh pr diff 99
```

Check CI status for a PR:

```bash
gh pr checks 99
```

Extract files changed in a PR:

```bash
gh pr view 99 --json files --jq '.files[].path'
```

Get the number of additions/deletions:

```bash
gh pr view 99 --json additions,deletions --jq '"additions: \(.additions), deletions: \(.deletions)"'
```

---

## Modifying Issues

Add a label to an issue:

```bash
gh issue edit 42 --add-label "needs-info"
```

Remove a label from an issue:

```bash
gh issue edit 42 --remove-label "triage"
```

Replace all labels on an issue:

```bash
gh issue edit 42 --label "bug,confirmed"
```

Post a comment on an issue:

```bash
gh issue comment 42 --body "Thanks for the report. Could you share a minimal reproduction case?"
```

Close an issue with a reason:

```bash
# Valid reasons: completed, not-planned
gh issue close 42 --reason "not-planned" --comment "Closing as out of scope — see #43 for the roadmap discussion."
gh issue close 42 --reason "completed" --comment "Fixed in v1.2.0."
```

---

## Modifying Pull Requests

Post a review comment on a PR (without approving or requesting changes):

```bash
gh pr review 99 --comment --body "Left a few notes inline. Main concern is the missing test coverage on the error path."
```

Request changes on a PR:

```bash
gh pr review 99 --request-changes --body "Please add tests for the edge cases noted above before this can merge."
```

> **NEVER approve or merge PRs.** MStack is an assistant tool — all merge decisions belong to the human maintainer. Do not use `gh pr review --approve` or `gh pr merge`.

Add a label to a PR:

```bash
gh pr edit 99 --add-label "needs-tests"
```

---

## Release Management

List existing tags:

```bash
git tag --sort=-version:refname | head -10
```

Get the most recent tag:

```bash
git describe --tags --abbrev=0
```

List commits since the last tag:

```bash
LAST_TAG=$(git describe --tags --abbrev=0)
git log "${LAST_TAG}..HEAD" --oneline
```

List commits since the last tag with full detail:

```bash
LAST_TAG=$(git describe --tags --abbrev=0)
git log "${LAST_TAG}..HEAD" --pretty=format:"%H %s" --no-merges
```

List merged PRs since the last tag (requires the merge commit messages to reference PR numbers, or use the API):

```bash
LAST_TAG=$(git describe --tags --abbrev=0)
SINCE_DATE=$(git log -1 --format=%aI "${LAST_TAG}")
gh pr list --state merged \
  --json number,title,author,mergedAt \
  --jq --arg since "$SINCE_DATE" '[.[] | select(.mergedAt > $since)]'
```

Create a release tag and push it:

```bash
git tag -a v1.2.0 -m "Release v1.2.0"
git push origin v1.2.0
```

Create a GitHub release from a tag:

```bash
gh release create v1.2.0 \
  --title "v1.2.0" \
  --notes "$(cat CHANGELOG_FRAGMENT.md)" \
  --latest
```

Create a pre-release:

```bash
gh release create v1.2.0-rc.1 \
  --title "v1.2.0-rc.1" \
  --notes "Release candidate — please test before the stable release." \
  --prerelease
```

---

## Rate Limiting

Check current GitHub API rate limit before bulk operations:

```bash
gh api rate_limit
```

Extract remaining core requests:

```bash
gh api rate_limit --jq '.resources.core.remaining'
```

Check and warn if low:

```bash
REMAINING=$(gh api rate_limit --jq '.resources.core.remaining')
if [ "$REMAINING" -lt 100 ]; then
  RESET=$(gh api rate_limit --jq '.resources.core.reset')
  echo "WARNING: Only $REMAINING API requests remaining. Resets at $(date -r $RESET 2>/dev/null || date -d @$RESET)."
fi
```

> Always check rate limits before running bulk issue or PR scans. If remaining < 100, pause and inform the user.

---

## Author Info

Fetch full profile for a GitHub user:

```bash
gh api "users/<USERNAME>"
```

Check if a user is a bot (type field):

```bash
gh api "users/somebot" --jq '.type'
# Returns "Bot" for bot accounts, "User" for humans
```

Get account age signals:

```bash
gh api "users/<USERNAME>" --jq '{login: .login, type: .type, created_at: .created_at, public_repos: .public_repos, followers: .followers}'
```

Example — compute account age in days from shell:

```bash
CREATED=$(gh api "users/<USERNAME>" --jq '.created_at')
CREATED_TS=$(date -j -f "%Y-%m-%dT%H:%M:%SZ" "$CREATED" +%s 2>/dev/null \
             || date -d "$CREATED" +%s)
NOW=$(date +%s)
AGE_DAYS=$(( (NOW - CREATED_TS) / 86400 ))
echo "Account age: $AGE_DAYS days"
```
