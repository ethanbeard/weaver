# RESOLVER.md — routing table

This is the dispatcher. When the human says X, you invoke Y. Check here
first before reaching for a tool directly. **Don't call `CronCreate` for
anything recurring** — that bypasses the loops registry and expires in
7 days. **Don't hand-roll skills** — use `/import-skill` to fetch and
adapt existing ones.

## Session-start checks (CLAUDE.md enforces these)

Before responding to the human on any fresh session:

1. `Read .claude/.onboarded` → if file missing, invoke `/onboard` (idempotent skill that runs any unfinished ritual stages)
2. `Read memory/<today>.md` → if exists, absorb as context from earlier today
3. `/loops sync` → reconcile the loop registry (also bootstraps the keepalive on first run)

Skip any of these three and you'll miss something load-bearing. They all use `Read` / tool calls; don't guess outcomes from context.

## When the human says...

### Scheduling and ambient automation

| Trigger | → |
|---|---|
| "Set up a morning briefing" / "check my X every Y" / "I want a daily thing" | `/loops add` |
| "What's running periodically?" / "show my automations" / "what does my agent do on a schedule?" | `/loops list` |
| "Turn off the heartbeat" / "stop loop X" | `/loops disable <name>` |
| "Turn on the heartbeat" / "enable loop X" | `/loops enable <name>` |
| "Something's off with my automations, force a refresh" | `/loops sync` |
| "Remind me at 3pm" / "in 45 min check Y" (one-off, not recurring) | `CronCreate` with `recurring: false` — auto-deletes after firing |

**Never call `CronCreate` directly for anything recurring.** Use `/loops add` so the task lands in the registry and gets renewed past the 7-day expiry.

### Phone and remote access

| Trigger | → |
|---|---|
| "Let me reach you from my phone" / "pair my iPhone" / "set up iOS access" | Built-in `/remote-control` — tell the human to type it in the chat input |
| "Keep remote control alive across reboots on my dedicated machine" (macOS, terminal CLI) | `/install-launchagent` |
| "Let me text you from Messages" / "SMS-style bridge" (CLI only, different UX from Remote Control) | `/install-imessage-channel` |

### Discovery and extension

| Trigger | → |
|---|---|
| "What can you do on this machine?" / "suggest useful setups" / "give me a tour" | Answer conversationally — mention what's actually wired up (check `mcp__*` tool prefixes, available skills, installed connectors). No skill for this; it's just a conversation. |
| "Grab the X skill from OpenClaw" / "import a skill from GitHub" / "port this skill" | `/import-skill <source>` |

### Utilities

| Trigger | → |
|---|---|
| Weather, temperature, forecast | `/weather` |

### First-run ritual

| Trigger | → |
|---|---|
| `.claude/.onboarded` missing at session start | Invoke `/onboard`. The skill is idempotent — inspects state and runs only missing stages. Do not improvise an alternative. |
| Human says "redo my onboarding" / "let's start over" / "change my name" | `/onboard <phase>` where phase is `persona`, `capabilities`, or `pairing`. Respect existing data unless they explicitly say to overwrite. |

## Cross-cutting rules

Every skill and every interaction is also governed by:

- **`.claude/_output-rules.md`** — output quality standards (no slop, exact phrasing preservation, deterministic links, title quality). Apply to chat output AND file writes.
- **Persona edits require disclosure.** When updating `IDENTITY.md` / `SOUL.md` / `USER.md` / `TOOLS.md` / `MEMORY.md`, tell the human what you changed and why. Surprising edits erode trust.
- **Prefer editing over creating.** Before writing a new file, check whether something similar already exists.

## Disambiguation

When multiple routes could apply:

1. **Prefer the most specific.** If `/import-skill` handles the intent, don't hand-roll a SKILL.md.
2. **Within BOOTSTRAP.md ritual, BOOTSTRAP wins.** For the first session only, the ritual's flow supersedes general resolver routing.
3. **When genuinely ambiguous, ask.** A clarifying question beats a wrong skill invocation.

## When to skip this table

- **One-off reminders** (fire once, no recurrence) — use `CronCreate` with `recurring: false` directly. The loops registry only manages recurring prompts.
- **Built-in slash commands** (`/remote-control`, `/config`, `/rename`, `/clear`, `/compact`, etc.) — these are Claude Code harness commands, not workspace skills. They don't route through here. Tell the human to type them.
- **Things outside the workspace scope entirely** — e.g., the human asks you to write code unrelated to the personal-assistant persona. Do the work directly; the resolver is for persona-shaped tasks.
