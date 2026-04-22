# Bot and Spam Detection Heuristics

These heuristics help identify automated, low-effort, or coordinated submissions. They are inputs for the maintainer — not automatic actions.

---

## Account Signals

**Treat these as weak signals. No single signal is disqualifying.**

| Signal | Notes |
|--------|-------|
| Account age < 7 days | New accounts are common among first-time contributors, but also common for spam campaigns. |
| `type` field is `"Bot"` | GitHub marks bot accounts explicitly. Cross-check with `gh api "users/<USERNAME>" --jq '.type'`. |
| Zero public repos AND zero followers | Suggests a throwaway account. A real new contributor usually has at least one forked repo. |
| Bulk submissions | Same account filing 5+ issues or PRs within a short window, especially if they are similar in structure. |
| Username is a random string or follows a pattern | e.g., `user1738492`, `contributor_abc123` — common in coordinated spam. |
| Profile has no bio, no avatar, no pinned repos | Alone this is meaningless, but in combination with other signals it adds weight. |

---

## Content Signals

**Focus on whether the content is genuinely useful, regardless of who wrote it.**

| Signal | Notes |
|--------|-------|
| Generic or template-like language | Phrases like "I found a bug in your excellent project" with no specifics. |
| Mismatched domain | The issue or PR topic does not match the repository at all (e.g., a CSS framework receiving a report about a Python data pipeline). |
| No reproduction steps for a bug report | Legitimate bugs should be reproducible. An issue with only "this doesn't work" and no version, environment, or steps is incomplete regardless of author. |
| No description, only a title | May be haste, may be spam. Ask for more detail before acting. |
| AI-generated markers | Overly formal or verbose prose, bullet lists for every sentence, unnecessary hedging ("It seems like perhaps this might be…"), repeated affirmations before answering. Note: this alone does not mean the content is invalid — AI-assisted contributions can be legitimate. |
| Copy-paste from another issue without adaptation | Check if the body is near-identical to an issue on another repo. |

---

## PR-Specific Signals

| Signal | Notes |
|--------|-------|
| Trivial change dressed up as a feature | e.g., a single-character whitespace fix submitted with a lengthy description of "improving code quality and maintainability." |
| Generated code with no tests | Especially suspicious when the PR touches core logic and adds no coverage. |
| Formatting-only changes | Unsolicited reformatting (tabs to spaces, trailing whitespace removal) with no functional change. These can cause merge conflicts and are rarely welcome unless the project has explicitly requested them. |
| PR touches files unrelated to the stated purpose | A "fix typo in README" PR that also modifies source files. |
| No linked issue | Not a hard rule, but for non-trivial changes it is worth noting. Many projects require an issue before a PR. |
| Branch name is `patch-1`, `patch-2`, etc. | The GitHub UI default — common among contributors who did not read `CONTRIBUTING.md`, but also common among low-effort submissions. |

---

## What This Is NOT

These heuristics exist to surface patterns for a maintainer to evaluate. They are not a rejection filter.

- **Not anti-new-contributor.** First-time contributors deserve a welcome and clear guidance. If an issue or PR is incomplete, the right response is to ask for more information, not to close it.
- **Not auto-reject.** No combination of signals automatically triggers a close or a spam label. Every detection is a prompt for the maintainer to look more carefully.
- **Not a judgment of the person.** A flagged submission is a submission that warrants closer review. The content may still be valid and valuable.
- **Not a substitute for reading the contribution.** Always read the actual issue or PR before acting on any signal.

When in doubt, ask a clarifying question. A real contributor will answer. A bot or spammer usually will not.
