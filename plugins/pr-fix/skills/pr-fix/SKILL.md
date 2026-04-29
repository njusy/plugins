---
name: pr-fix
description: Apply fixes requested via a PR review submission or a top-level PR comment. Use this skill when the user invokes /pr-fix or when invoked via the claude-pr-comment workflow with a Trigger field in the prompt. Always use this skill when the user says "pr-fix", "fix pr comments", "/pr-fix", or similar.
---

# PR Comment Fix

Workflow for applying fixes requested via a PR review or a top-level PR comment.

> **Network failures:** For any git or gh CLI command that contacts the network
> (git pull, git push, gh pr comment, etc.), retry up to 3 times with a 5-second
> delay between attempts before giving up.

## Step 1 — Read the PR

```bash
gh pr view <pr_number> --repo <repo>
gh pr diff <pr_number> --repo <repo>
```

Read the full PR description and diff before proceeding.

## Step 2 — Check out the PR branch

```bash
gh pr checkout <pr_number> --repo <repo>
```

## Step 3 — Understand what needs to be fixed

The approach depends on the `Trigger` value in the prompt:

### Trigger: `review_submitted`

Fetch all inline comments from the review:

```bash
gh api repos/<repo>/pulls/<pr_number>/reviews/<review_id>/comments
```

Each inline comment has a `path` (file), `line`, `original_line`, and `body` describing what needs to change. Read each referenced file at the indicated line and understand the requested fix. Collect all requested changes — they will all be applied in a single commit.

If any inline comment is ambiguous, post a PR comment asking for clarification and stop — do not proceed to Step 4.

### Trigger: `pr_comment`

The `Comment body` in the prompt describes what needs to change. Read the relevant files and locate the code to fix. If the request is ambiguous, post a PR comment asking for clarification and stop.

## Step 4 — Read CLAUDE.md and apply conventions

Read the repository's CLAUDE.md (if present) before writing any code:

```bash
cat CLAUDE.md 2>/dev/null || true
```

Apply all conventions, patterns, and instructions found there throughout the remaining steps.

## Step 5 — Implement the fix

Read relevant existing code before writing. Keep changes minimal and focused on the requested fixes.

## Step 6 — Run linter and tests

After implementing, run the linter and test suite as specified in CLAUDE.md (or the project's standard commands). If tests fail, fix them and re-run. **After 3 failed fix attempts, stop:** push the current branch and post a PR comment describing what is broken and why further automated fixing was abandoned:

```bash
gh pr comment <pr_number> --repo <repo> --body "$(cat <<'EOF'
Automated fix stopped after 3 attempts: <one sentence describing the failing test/lint issue and what was tried>.
EOF
)"
```

Then stop — do not push an additional commit beyond what was already staged.

## Step 7 — Commit

Stage only the files changed for this fix.

```bash
git add <files>
git commit -m "<pr_number>-fix: one sentence message"
```

Message format uses `<pr_number>-fix:` prefix since there is no issue number.

## Step 8 — Push

```bash
git push
```

The branch is already tracked. Retry up to 3 times with a 5-second delay on network failure.

## Step 9 — Acknowledge the fix

### Trigger: `review_submitted`

Reply to the submitted review by posting a top-level PR comment summarising what was fixed:

```bash
gh pr comment <pr_number> --repo <repo> --body "$(cat <<'EOF'
Fixed in <commit_sha>: <one sentence per inline comment addressed>.
EOF
)"
```

### Trigger: `pr_comment`

Post a threaded reply on the original comment:

```bash
gh api repos/<repo>/issues/comments/<comment_id>/replies \
  --method POST \
  --field body="Fixed in <commit_sha>: <one sentence describing what was changed>."
```

Use the `Comment ID` value from the prompt as `<comment_id>`.
