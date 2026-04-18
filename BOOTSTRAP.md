# BOOTSTRAP.md — first-run ritual (documentation)

This file describes the first-run experience for a fresh Weaver workspace.
It is **not** executed by the agent. The actual ritual lives at
`.claude/skills/onboard/SKILL.md` and is invoked via `/onboard`.

If you're reading this as a human new to the repo, here's what happens on
your first session.

## What triggers the ritual

`CLAUDE.md` has a session-start check that reads `.claude/.onboarded`.
If that file is missing, the agent silently invokes `/onboard` on the
first turn, before responding to you. The skill is idempotent — it
inspects workspace state and only runs the stages that aren't done yet.
So a partially completed ritual resumes where it left off on the next
session; nothing self-destructs or corrupts.

## The stages

### Stage 0 — persona + user

The agent introduces itself (it hasn't been named yet) and you decide
together:

- What the agent's name is
- What kind of creature it is (assistant, ghost, familiar, etc.)
- Its vibe and signature emoji
- Your name and what to call you

The agent writes these into `IDENTITY.md` and `USER.md`. `SOUL.md`
already has sensible defaults.

### Stage 1 — capability offer

Once the persona is established, the agent offers to take on recurring
work for you — morning briefings, pre-meeting prep, email triage, etc.
You can accept specifics, ask for ideas, decline, or say "hold on, I'm
going to install more tools first." All branches close the stage; none
stall.

A marker file (`.claude/.capability-offered`) gets written so this
stage doesn't re-fire on future sessions.

### Stage 2 — close out

The agent writes `.claude/.onboarded` with a timestamp. That flag is
the authoritative signal that onboarding is done; future sessions
skip the ritual when it's present.

### Stage 3 — phone pairing postscript (optional, last)

If `~/.claude/settings.json` doesn't already have
`remoteControlAtStartup: true`, the agent offers iOS pairing via
`/remote-control` as the final postscript. The agent doesn't get a
turn back after `/remote-control` fires — that's why it's last.

## Replaying the ritual

Run `/onboard` directly at any time to inspect state and resume any
missing stage. Pass a phase hint to force re-run:

- `/onboard persona` — redo stage 0 (agent will ask before overwriting
  existing IDENTITY/USER)
- `/onboard capabilities` — re-offer stage 1
- `/onboard pairing` — jump to stage 3

To fully replay onboarding from scratch:

```bash
rm -f .claude/.onboarded .claude/.capability-offered
# optionally: clear IDENTITY.md and USER.md back to templates
```

Next session, the agent detects the missing flag and reruns the ritual.

## Why a skill instead of a self-deleting file

Earlier versions of this workspace had the ritual as a Markdown file at
the workspace root that the agent was supposed to delete after running.
That design was brittle — partial completion left the file on disk,
causing the ritual to re-fire on the next session, often with confused
state. The skill + flag file approach separates "is onboarding done?"
(a boolean) from "how is onboarding done?" (a reusable skill).
