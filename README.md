# njusy-plugins

Claude Code plugins for the Njusy project.

## Plugins

### `tasks-export`

Distills a Claude Code session into a minimal, prioritized task list and saves it as structured JSON to `~/tasks/<repo>/`.

**Usage:** `/tasks-export`

**What it does:**
1. Re-reads the session and extracts concrete, actionable tasks (one concern per task)
2. Applies TDD splitting — test task first, implement task second, linked by `depends_on`
3. Displays the task table for confirmation
4. Saves the confirmed list to `~/tasks/<repo_name>/<dd-mm-yyyy>-tasks.json`

---

### `implement-task`

Implements a GitHub issue end-to-end: reads the issue, creates a branch, writes code, commits, and opens a PR.

**Usage:** `/implement-task <issue-number>` or `/implement-task` with a description in the prompt

**What it does:**
1. Reads the GitHub issue (or uses the description you provide)
2. Syncs `main` and creates a branch (`<username>/<number>-short-name`)
3. Implements the change following CLAUDE.md conventions
4. Commits and pushes, then opens a PR with `Closes #<number>`
5. Captures non-obvious learnings into CLAUDE.md or memory files after PR review

If the issue is unclear, it posts a comment with questions and stops — it never makes file changes on an ambiguous spec.

## Structure

```
.claude-plugin/
  marketplace.json          ← registry of all plugins in this repo
plugins/
  <plugin-name>/
    .claude-plugin/
      plugin.json           ← plugin metadata (name, version, description, author)
    skills/
      <skill-name>/
        SKILL.md            ← skill definition loaded by Claude Code
        [template.json]     ← optional supporting files referenced by the skill
```

---

## Working with plugins

### Adding this marketplace to Claude Code

Register this repo as a marketplace once — Claude Code will then let you browse and install any plugin from it:

```
/plugin marketplace add njusy/plugins
```

Verify it was added:

```
/plugin marketplace list
```

### Installing a plugin

After adding the marketplace, install any plugin by name:

```
/plugin install tasks-export@njusy-plugins
/plugin install implement-task@njusy-plugins
```

By default this installs at **user scope** (available in all your projects). To pin it to a specific repo instead:

```bash
claude plugin install tasks-export@njusy-plugins --scope project   # shared with all collaborators via .claude/settings.json
claude plugin install tasks-export@njusy-plugins --scope local     # only you, not committed
```

Then activate without restarting:

```
/reload-plugins
```

Skills are then available as slash commands (e.g. `/tasks-export`, `/implement-task`).

### Managing installed plugins

```
/plugin                                          # open interactive UI (Discover / Installed / Marketplaces / Errors)
/plugin disable tasks-export@njusy-plugins       # disable without uninstalling
/plugin enable  tasks-export@njusy-plugins       # re-enable
/plugin uninstall tasks-export@njusy-plugins     # remove completely
```

### Creating a new plugin

1. **Scaffold the directory structure:**

   ```bash
   mkdir -p plugins/<plugin-name>/.claude-plugin
   mkdir -p plugins/<plugin-name>/skills/<skill-name>
   ```

2. **Write `plugins/<plugin-name>/.claude-plugin/plugin.json`:**

   ```json
   {
     "name": "<plugin-name>",
     "version": "1.0.0",
     "description": "One-line description of what this plugin does.",
     "author": {
       "name": "<github-username>",
       "email": "<email>"
     },
     "license": "MIT",
     "keywords": ["tag1", "tag2"]
   }
   ```

3. **Write `plugins/<plugin-name>/skills/<skill-name>/SKILL.md`:**

   The frontmatter controls how Claude Code discovers and invokes the skill:

   ```markdown
   ---
   name: <skill-name>
   description: One sentence — when Claude should use this skill. Include the slash command name and natural-language triggers.
   allowed-tools: Bash Read Write   # optional: restrict which tools the skill may use
   disable-model-invocation: true   # optional: skip LLM calls, run shell blocks directly
   effort: low | medium | high      # optional: hint for resource planning
   ---

   # Skill title

   Skill instructions here...
   ```

   **Tips for a good `description` field:**
   - Name the slash command explicitly: `"Use when the user invokes /my-skill or says 'do X'."` — this is what Claude Code matches against.
   - List natural-language synonyms so the skill triggers reliably without the slash command.
   - Keep it to one sentence; the body of SKILL.md is where detail lives.

4. **Register in the marketplace** — add an entry to `.claude-plugin/marketplace.json`:

   ```json
   {
     "name": "<plugin-name>",
     "source": "./plugins/<plugin-name>",
     "description": "Same one-line description as plugin.json.",
     "version": "1.0.0",
     "author": { "name": "<github-username>" },
     "keywords": ["tag1", "tag2"]
   }
   ```

### Updating a plugin

- Bump `version` in both `plugin.json` and the marketplace entry when making breaking changes.
- Keep the `name` field in SKILL.md frontmatter stable — renaming it breaks any project that already uses the slash command.
- After pushing changes, users run `/plugin marketplace update njusy-plugins` then `/reload-plugins` to pick them up.

### Supporting files

Place any files the skill reads at runtime (JSON schemas, templates, prompt fragments) alongside SKILL.md in `skills/<skill-name>/`. Reference them by relative path from within the skill, e.g. `skills/tasks-export/template.json`.
