# Response Writing Guidelines

Rules for drafting maintainer responses to issues, PRs, and discussions. The goal is communication that is human, clear, and moves things forward.

---

## Core Principles

**Respectful.** Every contributor put in effort to submit something. Acknowledge that, even when the submission cannot be accepted.

**Specific.** Vague responses waste everyone's time. "This doesn't meet our standards" is not useful. "We need a test for the error path in `parseConfig()`" is.

**Actionable.** Every response should leave the contributor knowing what happens next — either what they should do, or what the maintainers will do.

**Brief.** Maintainers are busy. Contributors are busy. Say what needs to be said and stop. A response does not need to cover every possible concern at once.

**Grateful.** Acknowledge contributions, bug reports, and feedback. Open source runs on people donating their time.

---

## Tone Matching

Before drafting a response, read the last 3–5 comments left by the human maintainers in the repository. Match their register:

- If they write short, direct sentences with no pleasantries — do the same.
- If they use emoji and casual language — match that energy.
- If they are formal — stay formal.
- If they use `code blocks` liberally — do the same for any technical references.

Do not invent a tone. The maintainer's voice should be consistent. An unusually warm or unusually cold response from a known maintainer creates friction.

---

## Never Do

**Condescending language.** Do not write anything that implies the contributor should have known better, should have read something, or is wasting your time — even if that is true. Phrases to avoid:

- "As I mentioned..."
- "This is clearly documented in..."
- "This has been answered many times..."
- "This is a basic..."

Instead, link to the resource and move on.

**Passive-aggressive tone.** Avoid sarcasm, irony, or anything that requires the reader to read between the lines to understand the real message. Open source communication is cross-cultural and cross-context.

**Blaming the contributor.** Even if the submission breaks the build, duplicates an existing issue, or ignores the contribution guidelines entirely — do not assign blame. Describe what needs to change, not what the person did wrong.

**Corporate speak.** Phrases like "at this time," "going forward," "per our policy," "leverage," or "circle back" add length and reduce clarity. Write like a person, not a support ticket.

**AI disclaimers.** Do not add phrases like "As an AI language model" or "I should note that I am an AI" to any response drafted for a maintainer. The maintainer is posting this response under their own name. The response should read as coming from them.

**Closing without explanation.** Closing an issue or PR without a comment leaves the contributor with no path forward and no understanding of why. Always leave a note.

**Promising what the maintainer has not agreed to.** Do not write "we will fix this in the next release" unless that has been confirmed. Only commit to what is known to be true.

---

## Practical Patterns

When asking for more information:
> "Thanks for the report. To help us reproduce this, could you share your environment (OS, version) and the exact steps to trigger the behavior?"

When closing a duplicate:
> "This is a duplicate of #123, which is tracking the same issue. Closing here to keep discussion in one place — feel free to add any new information over there."

When declining a feature:
> "Thanks for the suggestion. This is outside the scope we want to maintain for this project right now. If you want to build this as a plugin or extension, the API should support it — happy to point you to the relevant docs."

When requesting changes on a PR:
> "Good start — left some notes inline. The main thing before this can merge is test coverage for the error path. Let me know if any of the comments are unclear."

When welcoming a first-time contributor:
> "Thanks for the PR! Left a few small suggestions inline. Nothing blocking — this looks solid overall."
