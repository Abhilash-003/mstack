---
name: mstack-release
version: 0.1.0
description: |
  Release automation: analyze merged PRs since the last tag, generate a
  changelog entry, determine the version bump, create a git tag, and publish
  a GitHub release. Human approves every destructive action.
  Use when asked to "ship a release", "cut a release", "create a release",
  "bump the version", or "publish a new version".
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

# Check if repo is configured
if [ -f "$PROJECT_ROOT/.mstack/config.yml" ]; then
  echo "REPO_CONFIGURED: true"
  REPO=$(grep "^repo:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "unknown")
  CHANGELOG_FORMAT=$(grep "changelog_format:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "keepachangelog")
  VERSION_SCHEME=$(grep "version_scheme:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "semver")
  RELEASE_BRANCH=$(grep "^branch:" "$PROJECT_ROOT/.mstack/config.yml" 2>/dev/null | awk '{print $2}' || echo "main")
  echo "REPO: $REPO"
  echo "CHANGELOG_FORMAT: $CHANGELOG_FORMAT"
  echo "VERSION_SCHEME: $VERSION_SCHEME"
  echo "RELEASE_BRANCH: $RELEASE_BRANCH"
else
  echo "REPO_CONFIGURED: false"
fi

# Get last tag and current branch
LAST_TAG=$(git describe --tags --abbrev=0 2>/dev/null || echo "none")
CURRENT_BRANCH=$(git rev-parse --abbrev-ref HEAD 2>/dev/null || echo "unknown")
echo "LAST_TAG: $LAST_TAG"
echo "CURRENT_BRANCH: $CURRENT_BRANCH"
```

If `GH_CLI` is not installed: stop and tell the user "MStack requires the GitHub CLI. Install from https://cli.github.com then re-run /mstack-release."

If `gh auth status` shows not authenticated: stop and tell the user "gh CLI is not authenticated. Run `gh auth login` first, then re-run /mstack-release."

If `REPO_CONFIGURED` is false: stop and tell the user "This repo is not configured for MStack yet. Run `/mstack-setup` first."

If `CURRENT_BRANCH` does not match `RELEASE_BRANCH`: warn the user "You are on branch `{CURRENT_BRANCH}`, but releases are expected from `{RELEASE_BRANCH}`. Proceeding anyway — confirm this is intentional before shipping."

---

## Step 1 — Analyze changes since last release

### If LAST_TAG is "none" (first release)

```bash
git log --oneline
```

Collect all commits in the repository history. Note that this is the first release — there is no previous tag to compare against.

### If LAST_TAG exists

```bash
# Commits since last tag
git log "$LAST_TAG"..HEAD --oneline

# Date of last tag for PR search
TAG_DATE=$(git log -1 --format="%aI" "$LAST_TAG" 2>/dev/null || echo "")
echo "TAG_DATE: $TAG_DATE"
```

Then fetch merged PRs since the tag date:

```bash
gh pr list --state merged \
  --search "merged:>$TAG_DATE" \
  --json number,title,author,mergedAt,labels,body \
  --limit 100
```

If `TAG_DATE` is empty, fall back to:

```bash
gh pr list --state merged \
  --json number,title,author,mergedAt,labels,body \
  --limit 50
```

Collect the commits and merged PRs. Store as `RAW_CHANGES`.

If `RAW_CHANGES` is empty (no commits and no merged PRs since last tag): tell the user "No changes found since `{LAST_TAG}`. Nothing to release." and stop.

---

## Step 2 — Categorize changes

For each commit message and PR title in `RAW_CHANGES`, assign it to one of the following categories:

| Category | Keywords / signals |
|----------|-------------------|
| `features` | `feat`, `feature`, `enhancement`, `add`, `implement`, `new`, `introduce` |
| `fixes` | `fix`, `bug`, `resolve`, `patch`, `correct`, `repair`, `revert` |
| `breaking` | `BREAKING`, `BREAKING CHANGE`, `!` suffix on a conventional-commit type (e.g. `feat!:`) |
| `docs` | `docs`, `doc`, `documentation`, `readme`, `changelog` |
| `internal` | `chore`, `refactor`, `ci`, `test`, `tests`, `build`, `deps`, `dependency`, `style`, `lint` |

Rules:
- A single commit/PR may match multiple categories if it contains multiple signals. Place it in the highest-priority category: `breaking` > `features` > `fixes` > `docs` > `internal`.
- If a PR has labels, use them to supplement keyword matching. Labels like `type: feature`, `enhancement`, `bug`, `breaking change`, `documentation` are strong signals.
- If no keyword matches, default to `internal`.
- PR body text containing "BREAKING CHANGE:" anywhere should always be classified as `breaking`.

Build four lists:
- `FEATURES` — list of `{number, title, author}` items (or commit hashes and summaries for commits without PRs)
- `FIXES` — same format
- `BREAKING_CHANGES` — same format; include the breaking change description from the PR body if present
- `DOCS` — same format
- `INTERNAL` — same format

Collect unique contributor logins from merged PRs. Store as `CONTRIBUTORS`.

---

## Step 3 — Determine version bump

Use `VERSION_SCHEME` (from config, default `semver`).

### semver rules

Parse `LAST_TAG` to extract `MAJOR.MINOR.PATCH`. Strip a leading `v` if present. If `LAST_TAG` is "none", there is no current version — proceed to first-release handling below.

Apply the highest-priority rule that matches:

| Rule | Condition |
|------|-----------|
| `major` bump | `BREAKING_CHANGES` list is non-empty |
| `minor` bump | `FEATURES` list is non-empty (and no breaking changes) |
| `patch` bump | Only `FIXES`, `DOCS`, or `INTERNAL` changes |

Compute `NEXT_VERSION`:
- major bump: `(MAJOR+1).0.0`
- minor bump: `MAJOR.(MINOR+1).0`
- patch bump: `MAJOR.MINOR.(PATCH+1)`

Prefix with `v`: `NEXT_TAG = "v{NEXT_VERSION}"`.

### First release (LAST_TAG is "none")

Ask the user:

> This appears to be the first release. What version should it be?
>
> A) v0.1.0 — initial pre-stable release
> B) v1.0.0 — initial stable release
> C) Enter a custom version

Record the chosen version as `NEXT_TAG`.

---

## Step 4 — Generate changelog entry

Read the existing `CHANGELOG.md` if it exists:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
CHANGELOG_PATH="$PROJECT_ROOT/CHANGELOG.md"
if [ -f "$CHANGELOG_PATH" ]; then
  echo "CHANGELOG_EXISTS: true"
  head -30 "$CHANGELOG_PATH"
else
  echo "CHANGELOG_EXISTS: false"
fi
```

Observe the existing format. If the file follows [Keep a Changelog](https://keepachangelog.com) conventions (`## [Unreleased]`, `## [x.y.z] - YYYY-MM-DD`), match that style exactly. If the file uses a different format, match what is already there.

Draft the changelog entry. Use this structure as the default (Keep a Changelog):

```
## [{NEXT_VERSION}] - {TODAY_DATE}

### Added
{For each item in FEATURES: "- {title} (#{number}, @{author})"}

### Fixed
{For each item in FIXES: "- {title} (#{number}, @{author})"}

### Changed
{For each item in DOCS: "- {title} (#{number}, @{author})"}

### Breaking Changes
{For each item in BREAKING_CHANGES: "- {title} (#{number}, @{author})\n  {breaking description if present}"}

### Contributors
{For each unique contributor: "@{login}"}
```

Omit sections that have no entries. If `CONTRIBUTORS` is empty (commit-only release), omit the Contributors section.

Store the draft as `CHANGELOG_ENTRY`.

---

## Step 5 — Present release plan and get approval

Display the following release plan to the user:

```
Release Plan
============================================
Current version : {LAST_TAG or "none (first release)"}
Next version    : {NEXT_TAG}
Tag name        : {NEXT_TAG}
Branch          : {CURRENT_BRANCH}
Bump type       : {major | minor | patch | first release}

Changes:
  Features       : {count of FEATURES}
  Bug fixes      : {count of FIXES}
  Breaking       : {count of BREAKING_CHANGES}
  Docs           : {count of DOCS}
  Internal       : {count of INTERNAL}

Contributors: {comma-separated list of @logins, or "none"}

--- Changelog draft ---
{CHANGELOG_ENTRY}
-----------------------
============================================
```

Then ask:

> How would you like to proceed?
>
> A) Ship it — execute the release as shown
> B) Change version — enter a different version number
> C) Edit changelog — revise the changelog entry before shipping
> D) Cancel — abort without making any changes

### Option B — Change version

Ask the user: "Enter the version number (e.g. `1.2.3` or `v1.2.3`):"

Normalize the input: strip leading `v`, then re-add it. Set `NEXT_TAG = "v{input}"`. Return to the release plan display with the updated version.

### Option C — Edit changelog

Show the current `CHANGELOG_ENTRY` and invite the user to provide a revised version. Replace `CHANGELOG_ENTRY` with the user's revision. Return to the release plan display with the updated entry.

### Option D — Cancel

Print:

```
Release cancelled. No changes made.
```

Stop.

---

## Step 6 — Execute release

Execute the following steps in order. Report progress after each step. If any step fails, stop immediately and report the error with the exact command that failed, so the user can recover manually.

### 6a. Update CHANGELOG.md

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
CHANGELOG_PATH="$PROJECT_ROOT/CHANGELOG.md"
```

If `CHANGELOG.md` exists and contains an `## [Unreleased]` section: replace that section header with `## [{NEXT_VERSION}] - {TODAY_DATE}`, preserving whatever content is under it, then insert a new empty `## [Unreleased]` section above it.

If `CHANGELOG.md` exists but has no `## [Unreleased]` section: prepend `CHANGELOG_ENTRY` after the top-level `# Changelog` heading (or at the very top if there is no such heading).

If `CHANGELOG.md` does not exist: create it with the following content:

```markdown
# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

{CHANGELOG_ENTRY}
```

After writing:

```bash
echo "CHANGELOG updated: $CHANGELOG_PATH"
```

### 6b. Update VERSION file

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
VERSION_PATH="$PROJECT_ROOT/VERSION"
```

If a `VERSION` file exists at the project root: overwrite it with `{NEXT_VERSION}\n` (the version without the `v` prefix, followed by a newline).

If no `VERSION` file exists: create one with `{NEXT_VERSION}\n`.

```bash
echo "VERSION updated: $VERSION_PATH"
```

### 6c. Commit the changes

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
cd "$PROJECT_ROOT"
git add CHANGELOG.md VERSION
git commit -m "chore: release {NEXT_TAG}"
echo "COMMIT: $(git rev-parse HEAD)"
```

### 6d. Create git tag

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
cd "$PROJECT_ROOT"
git tag -a "{NEXT_TAG}" -m "Release {NEXT_TAG}"
echo "TAG_CREATED: {NEXT_TAG}"
```

### 6e. Create GitHub release

Build the release body from `CHANGELOG_ENTRY`. Strip the `## [{NEXT_VERSION}] - {TODAY_DATE}` header line — the release title already carries the version.

```bash
gh release create "{NEXT_TAG}" \
  --title "{NEXT_TAG}" \
  --notes "{RELEASE_BODY}" \
  --target "{CURRENT_BRANCH}"
echo "RELEASE_URL: $(gh release view {NEXT_TAG} --json url --jq '.url')"
```

Store the output URL as `RELEASE_URL`.

### 6f. Ask about pushing

Ask the user:

> Release `{NEXT_TAG}` has been created locally. Ready to push the commit and tag to the remote?
>
> A) Push now — `git push origin {CURRENT_BRANCH} --tags`
> B) Push manually — I'll push when ready

If the user chooses **A**:

```bash
PROJECT_ROOT=$(git rev-parse --show-toplevel 2>/dev/null || pwd)
cd "$PROJECT_ROOT"
git push origin "{CURRENT_BRANCH}" --tags
echo "PUSHED: {CURRENT_BRANCH} + {NEXT_TAG}"
```

If the user chooses **B**:

```
To push manually, run:
  git push origin {CURRENT_BRANCH} --tags
```

---

## Step 7 — Summary

Print a final summary:

```
Release complete — {NEXT_TAG}
============================================
Version   : {NEXT_TAG}
Tag       : {NEXT_TAG}
Branch    : {CURRENT_BRANCH}
Changelog : {CHANGELOG_PATH}
Release   : {RELEASE_URL}
============================================
```

If the push was deferred, append:

```
Push pending. Run:
  git push origin {CURRENT_BRANCH} --tags
```

---

## Important rules

- **Never push without asking.** Always ask before `git push`. The user may want to review the commit locally or push from a different machine.
- **Never skip the approval step.** The release plan (Step 5) must always be shown and approved before any files are modified or commands run.
- **Fail loudly and stop.** If any git or gh command in Step 6 fails, stop immediately. Print the exact failing command and error. Do not attempt to continue — a partial release is worse than no release.
- **Preserve CHANGELOG format.** Read the existing CHANGELOG.md format before writing. Do not reformat or rewrite sections that are outside the new entry. Match the style already present.
- **Version normalization.** All git tags use the `v` prefix (e.g., `v1.2.3`). The `VERSION` file stores the bare version without the `v` prefix (e.g., `1.2.3`). Changelog headers use the bare version in brackets (e.g., `## [1.2.3] - 2026-04-13`).
- **First release default is v0.1.0.** When there is no previous tag, default to suggesting `v0.1.0` as the pre-stable release. Always ask the user to confirm.
- **Breaking changes trigger major bumps.** Even a single breaking change forces a major version bump. Do not suppress this — if the user disagrees, they can choose Option B in Step 5 to override the version.
