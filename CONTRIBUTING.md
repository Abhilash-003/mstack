# Contributing to MStack

Thanks for wanting to make MStack better. Whether you're fixing a prompt in a skill or adding an entirely new maintenance workflow, this guide gets you running.

## Quick start

```bash
git clone https://github.com/neuroanalog/mstack.git
cd mstack
./setup
```

MStack skills are Markdown files that Claude Code discovers from `~/.claude/skills/`. The setup script symlinks your repo's skill directories there, so edits take effect immediately.

## How to contribute

### Reporting bugs

Open a bug report with the skill name, steps to reproduce, and your environment (OS, Claude Code version, `gh` CLI version).

### Suggesting features

Open a feature request describing the maintenance workflow pain point and your proposed solution. Good candidates: new skill types, improvements to triage heuristics, additional response templates, better detection logic.

### Submitting changes

1. **Fork** the repository on GitHub
2. **Clone** your fork: `git clone https://github.com/YOUR-USERNAME/mstack.git`
3. **Create a branch**: `git checkout -b my-feature`
4. **Make your changes** (see conventions below)
5. **Test** your changes (see testing section)
6. **Commit** with a clear message (see commit style)
7. **Push** to your fork: `git push origin my-feature`
8. **Open a Pull Request** against `main`

For small fixes (typos, one-line changes), you can edit directly on GitHub and open a PR from there.

## How skills work

Each skill is a SKILL.md file with three parts:

1. **YAML frontmatter** — name, version, description, allowed tools
2. **Preamble bash** — checks gh CLI, auth status, repo configuration
3. **Prose workflow** — step-by-step instructions Claude follows

When a user types `/triage`, Claude reads `triage/SKILL.md` and follows it. That's the entire runtime. No compilation, no framework, no build step.

## Adding a new skill

1. Create a directory: `mkdir my-skill`
2. Create `my-skill/SKILL.md` with frontmatter:
   ```yaml
   ---
   name: my-skill
   version: 0.1.0
   description: |
     What this skill does. When to use it.
   allowed-tools:
     - Bash
     - Read
     - Write
     - AskUserQuestion
   ---
   ```
3. Add a preamble bash block (copy from any existing skill — they all use the same gh CLI check pattern)
4. Write the workflow in prose with bash blocks where needed
5. Add the directory name to the `SKILL_DIRS` list in `setup` (line with `SKILL_DIRS=`)
6. Run `./setup` to create the symlink

## Editing an existing skill

1. Edit the SKILL.md directly
2. Test by invoking the skill in Claude Code (e.g., `/triage`)
3. Changes are live immediately (symlinked)

## Conventions

**Follow GStack patterns.** MStack is modeled on [GStack's](https://github.com/garrytan/gstack) architecture. If you're unsure how to structure something, look at how GStack does it.

**Human checkpoints always.** Use AskUserQuestion before any action that affects GitHub: adding labels, posting comments, closing issues, creating tags, pushing. No exceptions. The human-in-the-loop design is the core of MStack — do not build skills that bypass it.

**Never auto-act on GitHub.** Skills analyze and draft. They never post, label, close, or push without explicit user confirmation. This is not optional. See ARCHITECTURE.md for why.

**Config goes in `~/.mstack/config.yaml`.** Use `bin/mstack-config get/set` to read/write. Flat keys only. Never store credentials in config — those stay in `gh`'s native auth store.

**Plumbing goes in `.mstack/`.** Skills write logs and reports to `.mstack/logs/` and `.mstack/reports/` at the project root. Use JSONL for structured records (one record per line). Use Markdown for human-readable reports. Always write logs even if the user takes no actions — the analysis record has value.

**Rate limit awareness.** All skills that make multiple API calls should check `RATE_REMAINING` in the preamble and warn if it drops below 50 during execution. Maintainers on large repos can hit limits quickly.

**Graceful degradation.** If an optional data source is unavailable (Dependabot 403, CI data missing), note it and continue. Never stop a full workflow because one check failed.

**Conservative bot detection.** When uncertain about bot or spam status, use a lower suspicion level. A false positive — flagging a real contributor — is worse than a false negative. Be fair to new contributors.

## Testing your changes

Since MStack is pure SKILL.md files, testing is straightforward:

1. **Frontmatter validation:**
   ```bash
   for f in */SKILL.md; do head -20 "$f" | grep "^name:" || echo "MISSING: $f"; done
   ```

2. **Config utility tests:**
   ```bash
   MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config set stale_days 60
   [ "$(MSTACK_STATE_DIR=/tmp/mstack-test bin/mstack-config get stale_days)" = "60" ] && echo PASS
   ```

3. **Setup syntax check:**
   ```bash
   bash -n setup && echo "setup: syntax OK"
   ```

4. **Integration test:** invoke each skill in Claude Code on a real repo (use a test repo, not production)

5. **Setup test:** run `./setup` on a clean machine or fresh directory

## Project structure

See CLAUDE.md for the full directory layout and state file reference.

## Commit style

One logical change per commit. Bisectable history. Examples:

- `Add /my-skill: one-sentence description`
- `Fix /triage: correct duplicate detection for closed issues`
- `Update templates/responses/needs-info: improve clarity`
- `Add response template: wont-fix scenario`

## What NOT to change

- **`_shared/response-guidelines.md` core principles** — the Never Do list and Core Principles are intentional. They reflect hard-won lessons about open source communication. Do not soften or remove them.
- **Human approval requirements** — do not add code paths that take GitHub actions without an AskUserQuestion checkpoint. This is non-negotiable.
- **Credentials** — never commit API keys, tokens, auth state, or anything that looks like a secret.

## Questions?

Open a [GitHub issue](https://github.com/neuroanalog/mstack/issues) — we're happy to help.
