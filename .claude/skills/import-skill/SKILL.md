---
name: import-skill
description: "Fetch a skill from a URL, GitHub path, or local directory and install it into this workspace (or the user's personal skills dir) as a Claude Code skill. Handles the OpenClaw → Claude Code port transformations automatically: strips OpenClaw-specific frontmatter metadata, adds a Prerequisites section for binary requirements, adapts references to OpenClaw-specific tools, and verifies the result. Use when the user says 'import the X skill', 'grab that skill from OpenClaw', 'port this skill', 'install a skill from github', or points at any external SKILL.md they want available here. Arguments: a source (URL, github shorthand like openclaw/weather, or local path) and optional --scope=project (default) or --scope=personal."
allowed-tools: WebFetch Read Write Bash(mkdir *) Bash(ls *) Bash(pwd)
argument-hint: "<source> [--scope=project|personal] [--name=<override>]"
---

# Import a skill from anywhere

Claude Code uses the [Agent Skills](https://agentskills.io) open standard.
OpenClaw uses the same standard. So do most community agent projects. This
skill fetches a `SKILL.md` (and any supporting files it references) and
installs it into a scope-appropriate location, applying the port rules
needed to make it work cleanly in Claude Code.

## Arguments

- `$ARGUMENTS[0]`: source — one of:
  - A **URL** to a raw `SKILL.md` file (e.g., a GitHub raw URL)
  - A **GitHub shorthand** in the form `openclaw/<skill-name>` — resolves to
    `https://raw.githubusercontent.com/openclaw/openclaw/main/skills/<skill-name>/SKILL.md`
  - A **full GitHub repo path** like `owner/repo/path/to/skill`
  - A **local path** to an existing skill directory
- `--scope=project` (default): install to `<cwd>/.claude/skills/<name>/`
- `--scope=personal`: install to `~/.claude/skills/<name>/`
- `--name=<name>`: override the skill directory name. Defaults to the skill's frontmatter `name` or the last path segment of the source.

Examples:

- `/import-skill openclaw/weather` — port OpenClaw's weather skill to project level
- `/import-skill openclaw/apple-reminders --scope=personal` — port apple-reminders to personal level
- `/import-skill https://raw.githubusercontent.com/someuser/claude-skills/main/my-skill/SKILL.md`
- `/import-skill ../some-local-skill-dir`

## Step 1: Resolve the source

Figure out what kind of source the user passed:

1. **If it starts with `openclaw/`**, it's OpenClaw shorthand. The URL is:
   ```
   https://raw.githubusercontent.com/openclaw/openclaw/main/skills/<skill-name>/SKILL.md
   ```
2. **If it starts with `http://` or `https://`**, it's a direct URL to a `SKILL.md` (or maybe a directory listing — try to fetch SKILL.md from the path directly).
3. **If it contains a `/` but no scheme, and has 2+ path segments**, assume it's a GitHub `owner/repo/path/to/SKILL.md` shorthand; rewrite to the raw URL.
4. **Otherwise**, treat it as a local path. Confirm it exists and contains a `SKILL.md`.

If the source is a URL and the user hasn't confirmed they trust it, stop and
ask: "This will fetch a skill from `<url>` and install it as a Claude Code
skill. Skills can contain instructions that affect agent behavior. Are you
sure you trust this source?"

## Step 2: Fetch the SKILL.md

- For URLs: use `WebFetch` with a prompt like "Return the raw SKILL.md content verbatim." Extract the markdown.
- For local paths: use `Read` on `<path>/SKILL.md`.

If the fetch fails or the result isn't a valid markdown file with YAML
frontmatter, stop and report the error to the user.

## Step 3: Parse frontmatter

Find the frontmatter block between the leading `---` markers. Extract:

- `name` — required. If missing, use the `--name` argument or the source's last path segment.
- `description` — required for Claude Code. If missing, fabricate one from the first paragraph of the body.
- Any `metadata` block, especially `metadata.openclaw`.
- Any other fields (`homepage`, `user-invocable`, `disable-model-invocation`, etc.)

## Step 4: Port the frontmatter

Apply these transformations:

### 4a. Drop the `metadata` block entirely

If the frontmatter has a `metadata: { openclaw: { ... } }` block (or any
other `metadata` block — it's an OpenClaw-specific nest, not a Claude Code
field), remove it. Save the following values from it first, because they'll
become content in the body:

- `metadata.openclaw.requires.bins` → list of binaries needed
- `metadata.openclaw.requires.env` → list of env vars needed
- `metadata.openclaw.os` → OS gating (darwin/linux/win32)
- `metadata.openclaw.install` → installer hints (brew formula, etc.)

### 4b. Keep what Claude Code understands

Claude Code recognizes these frontmatter fields:
`name`, `description`, `when_to_use`, `argument-hint`, `disable-model-invocation`, `user-invocable`, `allowed-tools`, `model`, `effort`, `context`, `agent`, `hooks`, `paths`, `shell`.

Preserve any that exist in the source.

### 4c. Infer `allowed-tools`

Scan the body for shell commands in code blocks. If you see `curl`, add
`Bash(curl *)`. If you see `gh`, add `Bash(gh *)`. And so on. This
pre-approves the commands the skill actually runs, so the user isn't
prompted on every invocation. Add the inferred list to frontmatter as
`allowed-tools: Bash(cmd1 *) Bash(cmd2 *)`.

If you're unsure what commands the skill uses, skip this — the user will
just get per-command prompts the first time.

### 4d. If the skill has side effects, consider `disable-model-invocation`

If the skill's body contains send-message, post-to-social, write-file,
delete, deploy, or payment operations, add `disable-model-invocation: true`
so the agent can't auto-trigger it. Tell the user you did this and why.

## Step 5: Port the body

Apply these transformations to the body content:

### 5a. Insert a Prerequisites section

If you saved binaries from step 4a, insert a new section right after the
main heading, before the first existing section:

```markdown
## Prerequisites

This skill requires the following binaries to be available on `PATH`:

- `<binary>` — <one-line description or install hint if you have one>

If any are missing, tell the user to install them before invoking this skill.
```

If you also saved install hints from `metadata.openclaw.install`, include
them in the section. For example, if `install` was a `brew` entry with
formula `steipete/tap/remindctl`, the hint becomes:
`Install with: brew install steipete/tap/remindctl`.

### 5b. Adapt OpenClaw-specific tool references

Search the body for these patterns and replace or flag:

| Old (OpenClaw) | New (Claude Code) |
|---|---|
| `cron` tool / `systemEvent` | `mcp__scheduled-tasks__create_scheduled_task` or `/schedule` routine |
| `NO_REPLY` / `no_reply` sentinel | just end the run silently; there's no silent-token in Claude Code |
| `HEARTBEAT_OK` | strip — this workspace ends heartbeat fires with an empty turn instead of echoing a sentinel string |
| `message` tool | replace with a reference to clawnet or the appropriate channel plugin |
| `canvas.*`, `node.invoke`, `sessions_*` | no direct equivalent — if the skill depends on these heavily, flag it for manual review |
| `openclaw.json` config | flag for manual review — no direct equivalent |
| ClawHub URLs (`clawhub.ai/...`) | leave as-is, they're reference links |

When you make a replacement, note it in a comment block at the bottom of the
ported file so the user can see what you changed:

```markdown
<!-- import-skill notes:
  - Replaced `cron` tool reference with mcp__scheduled-tasks__create_scheduled_task
  - Stripped metadata.openclaw.install (brew: curl)
-->
```

Block-level HTML comments are stripped from CLAUDE.md by the harness, but
they're preserved in skill content. So they stay visible if someone reads the
SKILL.md later.

### 5c. Handle `{baseDir}` substitutions

OpenClaw skills sometimes use `{baseDir}` to reference files bundled with the
skill. Claude Code's equivalent is `${CLAUDE_SKILL_DIR}`. Replace every
`{baseDir}` with `${CLAUDE_SKILL_DIR}`.

## Step 6: Write the ported skill

Figure out the target directory based on `--scope`:

- `project` (default): `<cwd>/.claude/skills/<name>/SKILL.md`
- `personal`: `~/.claude/skills/<name>/SKILL.md`

Create the directory (`mkdir -p`), then write the ported content.

If the target already exists, ask the user whether to overwrite or choose a
different name. Don't silently overwrite.

## Step 7: Handle supporting files

If the source is a GitHub URL or local directory (not just a single
SKILL.md), also fetch/copy any supporting files referenced from the main
SKILL.md (scripts, templates, example files). Use WebFetch or Read for each,
write to the target directory.

Don't port files blindly — only the ones the SKILL.md actually references.

## Step 8: Verify

- Read back the written SKILL.md.
- Confirm the frontmatter is valid YAML.
- Confirm the `name` matches the directory.
- Run `ls` on the target directory and show the user what got installed.

## Step 9: Report to the user

Tell the user:

1. ✅ Skill imported: `/<name>`
2. 📂 Installed to: `<target path>`
3. 🔧 Changes from source:
   - Listed each transformation (stripped metadata, added Prerequisites, adapted cron references, etc.)
4. 🧪 To try it: type `/<name>` or ask a question that matches its description
5. ⚠️ Flagged items needing manual review (if any): <list>

If the skill's description suggests it depends on something the user doesn't
obviously have (API keys, specific hardware), mention it as a caveat.

## Live change detection

Claude Code watches `.claude/skills/` for file changes and picks up new
skills in the current session — with one exception: creating the top-level
`.claude/skills/` directory for the first time in a session requires a
restart. If `.claude/skills/` didn't exist when the session started, tell
the user they may need to run `/clear` or restart Claude Code to see the new
skill in the slash command menu.
