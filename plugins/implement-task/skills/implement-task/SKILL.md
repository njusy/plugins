---
name: implement-task
description: Use this skill when the user invokes /implement-task or asks to implement a GitHub issue, work on a task/ticket, or start working on an issue by number. Always use this skill when the user says "implement task", "work on issue #N", "/implement-task", or similar.
---

# Implement Task

Follow all branch, commit, and PR conventions from CLAUDE.md.

## Step 0 — Detect current repo

```bash
REPO=$(gh repo view --json nameWithOwner --jq '.nameWithOwner')
```

Use `$REPO` for all `gh` commands throughout this workflow.

## Step 1 — Get the issue number

**If the user's prompt includes a full issue description or task specification (not just a number), skip Steps 1 and 2 entirely and use that description as the issue context.** Derive the issue title/scope from the prompt text and proceed to Step 3.

Otherwise, use provided issue number.

## Step 2 — Read the issue

> Skip this step if the issue description was provided directly in the prompt.

```bash
gh issue view <number> --repo $REPO
```

Read the full issue body, comments, and any linked context before proceeding.

> **Network failures:** For any git or gh CLI command that contacts the network
> (git pull, git push, gh pr create, etc.), retry up to 3 times with a 5-second
> delay between attempts before giving up.

> **STOP RULE — Unclear issue:**
> If the issue is unclear or you need clarification before you can implement it correctly,
> do **NOT** make any file changes and do **NOT** pause the session to ask questions interactively.
> Instead:
> 1. Post a comment on the issue with your questions:
>    ```bash
>    gh issue comment <number> --repo $REPO --body "$(cat <<'EOF'
>    I need clarification before I can implement this issue:
>
>    <your questions here>
>    EOF
>    )"
>    ```
> 2. End the session immediately. Do not proceed to Step 3.

## Step 3 — Sync main

```bash
git checkout main && git pull
```

## Step 4 — Create branch

Use the branch naming convention from CLAUDE.md. Derive the short name (2–4 lowercase hyphenated words) from the issue title.

```bash
git checkout -b <username>/<number>-short-name
```

To get the current git username: `git config user.name` (slugify if needed).

## Step 5 — Implement

Read relevant existing code before writing. Follow the architecture rules in CLAUDE.md. Keep changes minimal and focused on the issue scope.

### Test failures

If tests fail after your changes, attempt to fix them. After **3 failed fix attempts**, stop trying to fix the tests. Proceed to Step 6 and Step 7 as normal, then post a comment on the PR:

```bash
gh pr comment <pr-url> --body "$(cat <<'EOF'
⚠️ Tests are failing and could not be fixed after 3 attempts.

<brief description of what is failing and why>
EOF
)"
```

Do not continue retrying or making further changes to fix the tests.

## Step 6 — Commit

Stage only the files changed for this issue. Follow the commit message format from CLAUDE.md.

```bash
git add <files>
git commit -m "<number>: one sentence message"
```

## Step 7 — Push and open PR

```bash
git push -u origin <branch>
```

Follow the PR title format from CLAUDE.md. If there is an issue number, PR body must include `Closes #<number>`; otherwise omit that line.

```bash
gh pr create --title "<number>: short description" --body "$(cat <<'EOF'
## Summary
- <bullet points>

## Test plan
- [ ] <verification steps>

Closes #<number>

🤖 Generated with [Claude Code](https://claude.com/claude-code)
EOF
)"
```

## Step 7 — Capture learnings (only Sonnet and higher)

If you are running on haiku model then skip this step. Only when running on sonnet or higher model.
After the PR review is done, reflect on what happened during this implementation session. Ask: **did anything non-obvious come up that would help future sessions?**

Examples of things worth capturing:
- A pattern or convention discovered from reading existing code (e.g. how errors are wrapped, how tests are structured)
- A decision made that isn't obvious from the code (e.g. why a certain approach was chosen over another)
- A gotcha or edge case that caused confusion or required backtracking
- A new rule that emerged from PR review feedback

### What to update

- **CLAUDE.md** — for project-wide conventions, handler guidelines, architecture rules, or patterns that every future implementation should follow
- **Memory files** — for patterns specific to a sub-system, feedback about how to collaborate, or anything that isn't a hard project rule

If nothing non-obvious came up, skip this step — do not invent learnings.

### How to update CLAUDE.md

Add the learning under the most relevant existing section, or create a new section if needed. Keep entries concise (1–3 lines). Do not duplicate what is already documented.

```bash
# Read CLAUDE.md first, then Edit to add the new entry
```

### How to update memory

Use the Write tool to create or update the relevant memory file in the project's memory directory (check `~/.claude/projects/` for the matching project path), then update `MEMORY.md` with a pointer if it's a new file.
