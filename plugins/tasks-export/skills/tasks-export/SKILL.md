---
name: tasks-export
description: Runs the full pipeline — distills session into tasks, confirms with the user, then saves as JSON. Use when the user wants to do both steps at once.
when_to_use: Use when the user invokes `/tasks-export` or says "export tasks", "run tasks export".
disable-model-invocation: true
allowed-tools: Bash Read Write
effort: medium
---

# Tasks Export Skill

## Session context (auto-detected)

```!
echo "Today's date: $(date +%d-%m-%Y)"
ORIGIN=$(git remote get-url origin 2>/dev/null)
GIT_PROJECT=$(echo "$ORIGIN" | sed 's/.*\///; s/\.git//')
echo "Git repo name: ${GIT_PROJECT:-unknown — ask user}"
echo "~/tasks/ dir: $(test -d ~/tasks && echo 'exists' || echo 'will be created')"
```

## Purpose

Rethink the entire session conversation and distill it into a **minimal, prioritized list of small tasks** — one concern, one component, one change per task. Then save the confirmed list as a single JSON file to `~/tasks/<repo_name>/` for a Claude agent to implement.

If a task feels like it has two steps, split it.

## Output format

### Inline display

Print the distilled task list in the conversation. Do **not** write files during the distill phase.

```
## Tasks

| # | Title | Priority | Model | Depends on |
|---|-------|----------|-------|------------|
| 1 | Short human-readable name | high/medium/low | haiku/sonnet/opus | — or task # |
...

### Task 1 — <Title>
**Priority:** high | medium | low
**Model:** haiku | sonnet | opus
**Depends on:** none | Task N
**Description:** [Context line.] [Imperative body: what to implement, with exact project-relative paths.] [Patterns: see `<file>` for the existing pattern — omit if nothing comparable exists.] Out of scope: [what this task explicitly does NOT include]. Verify: [Checkpoint list — items the agent should confirm are present or correctly applied before finishing. No commands to run — tests, linting, and type checks are handled externally.]
```

### JSON file

One file: `~/tasks/<repo_name>/<dd-mm-yyyy>-tasks.json`

Read `skills/tasks-export/template.json` for the JSON schema and a worked example.

## Steps

### Phase 1 — Distill

1. **Re-read the session.** For each item ask:
   - What was decided? → becomes a task
   - What is broken or missing? → becomes a task
   - What was deferred or flagged for later? → lower-priority task
   - What is implicit but required? → prerequisite task
   Discard: opinions, exploratory tangents, questions answered inline, anything without a concrete outcome.
2. **Draft candidate tasks** (internal — do not print this).
3. **Prune and split**: remove duplicates; split any task whose title contains "and".
   One task = one concern. Prefer 3–8 tasks. Infrastructure before features.
4. **Apply TDD splitting**: for each task, decide if it warrants a unit test.
   - **Yes** (split into two): the task produces a concrete, deterministic artifact (file, function, API endpoint, schema) whose behaviour can be asserted without running an LLM or requiring heavy integration setup.
   - **No** (keep as one): pure configuration, infrastructure, prompt/skill authoring, or anything where writing a test has more overhead than value.
   When splitting, produce two tasks linked by `depends_on`:
   - **Test task** (order N): Write the test file(s) with every test marked as skipped/pending so the PR passes without implementation. Name exact test file paths. Model: haiku if a single file. Description must end with: "All tests must use the framework's skip/pending marker — do not implement the feature."
   - **Implement task** (order N+1): Implement the feature and remove the skip/pending markers. Depends on the test task. Description must include: "Enable the tests written in the previous task by removing skip/pending markers."
   Do not apply TDD splitting to the test task itself, infrastructure tasks, or tasks already flagged as low priority.
5. **Assign priority**: high = blocks other work; medium = important but not blocking; low = deferred/nice-to-have.
6. **Write self-contained descriptions.** For each task:
   - **Model** — `haiku` for a single named file with a clearly bounded change; `sonnet` for multi-file work or when an existing pattern must be understood first; **Default to `sonnet` when in doubt.**
   - **Context line** (first sentence of Description): only add for sonnet/opus when the agent genuinely needs to understand existing code first — `"Before implementing, read \`<dir-or-files>\` to understand <pattern>."` Omit for haiku tasks and any task where the file path and change are already explicit.
   - **Body** — imperative, no "as discussed" references. Name exact project-relative paths for every file to create or edit. State the target directory explicitly when creating new files.
   - **Patterns** — if a similar implementation exists, name its path: `"Patterns: see \`<file>\` for the existing pattern."` Omit if nothing comparable exists.
   - **Out of scope** — one sentence on what is explicitly NOT part of this task.
   - **Verify** — a checkpoint list the agent reviews before finishing: items to confirm exist, are wired up correctly, or match an expected structure. Do NOT include commands to run tests, linting, or type checks — those are handled by external scripts after the task completes.

### Phase 2 — Display & Confirm

7. **Print the task table** followed by the per-task detail blocks (see Inline display format above).
   Ask the user if the list looks right, and whether any tasks should be split, merged, reprioritized, or dropped. Incorporate any feedback before proceeding to Phase 3.

### Phase 3 — Save

8. **Verify explicitness** — For every task description, ensure it names:
   - Exact project-relative file paths (not vague references like "the user model").
   - Exact filenames for files to create or edit.
   - Target directory when creating new files.
   - What a successful result looks like (concrete artifact or verifiable behaviour).
   Rewrite vague descriptions before proceeding.
9. **Resolve repo_name**: use `$ARGUMENTS` if set; else use the git-detected name; else ask the user.
10. **Construct and save** the JSON using the schema in `skills/tasks-export/template.json`. Create `~/tasks/<repo_name>/` if needed. Save to `~/tasks/<repo_name>/<dd-mm-yyyy>-tasks.json`.
11. **Report** to the user: the saved file path and a summary table (title → model → priority → order → depends_on).

## Notes

- Do not invent tasks that were not discussed. Stick strictly to what emerged from the conversation.
- Do not include tasks for things already completed during the session.
- Write descriptions in the imperative ("Create...", "Add...", "Fix...", "Extract...").
- If the session produced no actionable tasks (e.g., pure Q&A), say so explicitly rather than fabricating work.
