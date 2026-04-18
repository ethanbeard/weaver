# Personal assistant workspace

## READ THIS FIRST — required session startup

On your first response in any fresh session (new session, after `/clear`,
or after `/compact` — not on plain resume), you MUST do the three checks
below using real tool calls, **before** responding to the human. Do not
guess outcomes from context. **Run the tools.**

### 1. Has onboarding been completed?

Call `Read` on `.claude/.onboarded` at the workspace root.

- If the file exists → onboarding is complete. Skip to step 2.
- If `Read` returns "file not found" → the ritual hasn't finished yet.
  Silently invoke the `/onboard` skill. It is idempotent: it inspects
  workspace state (IDENTITY.md, USER.md, `.claude/.capability-offered`)
  and runs only the stages that are missing. Let the skill drive the
  conversation from here — don't improvise your own onboarding.

Do not skip this check because "the workspace looks configured" or "it
probably ran before." The flag file is the only reliable signal.

### 2. Is there a journal entry for today?

Call `Read` on `memory/YYYY-MM-DD.md` where `YYYY-MM-DD` is today's date.

- If it exists → read it; it may contain context from earlier conversations
  today that affects how you respond.
- If it doesn't exist → skip; you'll create one later when something's
  worth journaling.

### 3. Sync scheduled loops

If `.claude/loops/` exists in this workspace, silently invoke `/loops sync`.
That skill reconciles the loop registry (files in `.claude/loops/*.md`)
with the currently-registered crons, and **it is how the keepalive loop
first gets registered on a brand-new install**. If you skip this step
on the first session after install, nothing ever gets scheduled and no
automation wakes up. Don't skip it.

`sync` is idempotent and silent by default — no output unless something
failed. It's cheap; calling it every session costs you one tool call.

If `.claude/loops/` doesn't exist at all, this workspace isn't using the
loops system; move on.

### Then respond

Once all three checks have run, proceed with the human's message. You
don't need to announce that you did the checks — just behave consistently
with what you found. If `/onboard` fired, the skill is driving the first
turn; follow its instructions. If the onboarded flag was present, respond
as your already-established persona.

**Common failure mode to avoid:** scanning the loaded persona files
(`IDENTITY.md` blank, `USER.md` blank, etc.), concluding "this is a fresh
workspace" on that evidence alone, and inventing your own onboarding. The
blank templates are a hint, but `.claude/.onboarded` is the authoritative
signal. Always confirm with `Read`.

---

## Always-loaded files

The imports below load at every session start (and after `/compact`). They
describe your persona, your human, how you operate, and the governance
layer you consult when routing tasks. All of them are editable by either
the human or you. When you edit one, tell the human what and why.

### Identity, relationship, conventions

@./IDENTITY.md
@./SOUL.md
@./USER.md
@./TOOLS.md

### Operating instructions

@./AGENTS.md

### Cross-cutting governance

@./.claude/RESOLVER.md
@./.claude/_output-rules.md

### Long-term memory

@./MEMORY.md

### Heartbeat checklist

@./HEARTBEAT.md

## Key files — read on-demand

Not auto-loaded. Go read them when the task calls for it.

- `BOOTSTRAP.md` — human-readable documentation of the first-run ritual. The ritual itself is implemented as the `/onboard` skill; this file explains what the skill does for someone reading the repo. Not used by the agent directly.
- `memory/YYYY-MM-DD.md` — today's daily journal. Read at session start; write to it when there's something worth remembering.
- `.claude/loops/*.md` — scheduled-loops registry. Each file is one loop. Managed via `/loops`; don't edit directly unless you're the `/loops` skill.
- `.claude/skills/<name>/SKILL.md` — skill implementations. Read when invoking a skill or debugging one.
- `.claude/launchd/` — plist + wrapper scripts, only present if `/install-launchagent` has been run.
- `.gitignore` — keeps `.DS_Store`, `.claude/settings.local.json`, and runtime logs out of version control.
