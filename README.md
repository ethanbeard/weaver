# Weaver

A personal-assistant pattern for Claude Code. Scheduled tasks — morning
briefings, calendar nudges, email triage, whatever — fire as new turns
**inside the chat you're already in**, not as separate sidebar sessions
or background daemons. Works the same in the desktop app and terminal
CLI.

## What's actually new

Claude Code already handles the easy parts: persistent persona via
`CLAUDE.md` imports, markdown memory, skills. Weaver bundles those into
a personal-assistant shape, but the bundle isn't the point.

**The load-bearing piece is `/loops`** — a file-based registry for
ambient scheduled prompts that fire as new turns in the *active* chat
(via `CronCreate`), plus a daily `keepalive` loop that renews
everything past Claude Code's 7-day session-scoped cron expiry. That's
what lets you have an ambient personal assistant without running a
daemon or a separate channels matrix.

Strip Weaver to its smallest interesting form and it's
`.claude/skills/loops/` plus `.claude/loops/keepalive.md`. The persona
files are skin on top.

## Prerequisites

Claude Code + a claude.ai subscription. Either surface works:

- **Desktop app** — [claude.ai/download](https://claude.ai/download)
- **Terminal CLI** — `brew install --cask claude-code` on macOS, or the [one-line installer](https://claude.ai/install.sh) on macOS/Linux/WSL

## Install

### 1. Make an empty directory and open Claude in it

```sh
mkdir ~/weaver
```

- **CLI**: `cd ~/weaver && claude`
- **Desktop**: **New session**, set **Project folder** to `~/weaver`

### 2. Paste this as your first message

```sh
Hello, world. Please clone `https://github.com/ethanbeard/weaver` into the current directory, then tell me to type `/clear` to reload the session.
```

The "Hello, world" opener seeds a pleasant auto-title on the session.
The agent handles the clone and tells you what to do next.

### 3. Type `/clear`, then say hi

`/clear` picks up the new `CLAUDE.md`. Send any message after and the
`/onboard` skill fires automatically — four short questions (your name,
the agent's name, its vibe, its emoji), plus optional capability setup
and phone pairing. Takes about a minute.

If `/clear` doesn't pick up the files, exit and reopen Claude in the
same directory.

## Ambient automation — the `/loops` system

The reason this project exists. Scheduled tasks fire as new turns
**inside the currently active chat**. If your session is paired with
the Claude iOS app via Remote Control, you see them on your phone too —
same chat, mirrored.

Each loop is a markdown file in `.claude/loops/` with YAML frontmatter
(cron, enabled, description) and a prompt body. Manage them via the
`/loops` skill — it wraps `CronCreate` so each loop lands in the
registry and survives the 7-day cron expiry.

```text
/loops              # list everything
/loops add          # add a new loop interactively
/loops enable foo   # turn on
/loops disable foo  # turn off
/loops remove foo   # delete
/loops sync         # reconcile registry with active crons
```

Two loops ship with the workspace:

- **`keepalive`** (enabled) — fires daily at 03:07 and runs `/loops sync`,
  renewing anything within 48 hours of expiring. Don't disable this
  unless you know what you're doing.
- **`heartbeat`** (disabled) — fires every 30 minutes and acts on
  whatever's in `HEARTBEAT.md`. Empty checklist = silent fire. Useful
  for batching periodic checks (mail, calendar, weather) into one
  cadence. Enable with `/loops enable heartbeat`.

Add your own by saying *"check my email every weekday at 8am and let
me know if anything's urgent"* — the agent invokes `/loops add` for
you.

## What's in here

| File | Purpose |
|---|---|
| `CLAUDE.md` | Entry point the harness loads. `@`-imports the rest. |
| `IDENTITY.md`, `SOUL.md`, `USER.md`, `TOOLS.md` | Who the agent is, who you are. Editable by either of you. |
| `AGENTS.md` | Operating instructions + session-startup checklist. |
| `MEMORY.md`, `memory/` | Curated long-term memory + daily journal. Agent-maintained. |
| `HEARTBEAT.md` | Checklist the `heartbeat` loop reads when enabled. |
| `BOOTSTRAP.md` | Human-readable docs for the first-run ritual. Not executed — see `.claude/skills/onboard/`. |
| `.claude/skills/` | `onboard`, `loops`, `import-skill`, `weather`, plus optional setup skills. |
| `.claude/loops/` | Scheduled-prompt registry. Managed via `/loops`. |
| `.claude/.onboarded` | Flag written when onboarding finishes. Remove to replay. |

Everything is markdown. Everything is editable by you or the agent.

## Optional extras

- **Phone access** — type `/remote-control` (built-in Claude Code
  command) to pair the iOS app with your current session. For
  auto-enable on new sessions, `/config` → **Enable Remote Control for
  all sessions**.
- **`/install-launchagent`** — for dedicated always-on machines (Mac
  mini) running the CLI with tmux. Not needed if you use the desktop
  app with **Open at login**.
- **`/install-imessage-channel`** — SMS-style bridge via iMessage.
  Terminal CLI only. Remote Control already covers "reach my agent
  from my phone" — this is for the Messages-app UX specifically.

## Adding skills

Ask the agent: *"Import the weather skill from `<url>`"* or point at any
SKILL.md. The `import-skill` skill fetches, ports, and installs at
project scope (`.claude/skills/`) or personal scope
(`~/.claude/skills/`).

Or write your own at `.claude/skills/<name>/SKILL.md`. See
[Anthropic's skills docs](https://code.claude.com/docs/en/skills).

## What's deliberately missing

- **No daemon, no gateway, no channels matrix.** `CronCreate` +
  `keepalive` replace all of it. The only "always-on" requirement is a
  Claude Code session being open somewhere.
- **No install script.** `mkdir`, open Claude, paste a paragraph.
- **No shell hooks, no `settings.json` changes.** Default Claude Code
  settings work.
- **No self-deleting files.** Onboarding is an idempotent skill gated
  on a flag file.
- **Minimal shipped skills.** One example (`weather`), the onboarding
  skill, the loops registry, and optional setup skills. Everything
  else is `/import-skill` on demand.

## License and attribution

Persona file templates (`AGENTS.md`, `SOUL.md`, `IDENTITY.md`,
`USER.md`, `TOOLS.md`, `HEARTBEAT.md`) are adapted from the
[OpenClaw](https://openclaw.ai) workspace templates (MIT). The
`CLAUDE.md` import structure, `/onboard` ritual, `/loops` registry,
and setup skills are original to this project.
