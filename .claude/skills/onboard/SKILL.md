---
name: onboard
description: Run (or resume) the first-run ritual for a fresh Weaver workspace — establish persona, capture user, offer capabilities, optional phone pairing. Idempotent — inspects workspace state and only runs the steps that haven't been done. Session-start check in CLAUDE.md invokes this automatically when `.claude/.onboarded` is missing. Humans can also invoke it manually to re-run any phase.
---

# /onboard — first-run ritual

The workspace's onboarding flow. Lives here as a skill (not a self-deleting
file at the workspace root) so it's:

- **Idempotent**: inspects state on each invocation, only runs missing steps
- **Recoverable**: partial completions don't leave dangling files; a crashed
  ritual resumes on next session
- **Replayable**: humans can run `/onboard` any time to redo a phase
- **Documented**: `BOOTSTRAP.md` at the workspace root describes the ritual
  for humans reading the repo

## When this runs

Automatically: `CLAUDE.md`'s session-start check tests for
`.claude/.onboarded`. If the file is absent, it silently invokes `/onboard`
on the first turn of the session, before any response to the human.

Manually: the human types `/onboard` to redo the ritual. Skill respects
existing state (doesn't wipe a filled-in `IDENTITY.md` unless the human
explicitly asks).

## Voice throughout

Short turns, no preamble, no throat-clearing. `.claude/_output-rules.md`
applies here too. The human will feel a drawn-out scripted onboarding;
don't give them one. Speak naturally, keep each message tight, and move
forward the moment a step resolves.

## State inspection (run first, before any human-facing output)

Check each of the following with actual tool calls. Don't guess.

1. **Persona** — Read `IDENTITY.md`. Is the `**Name:**` field still a
   template placeholder (contains `_(pick something you like)_` or is
   just blank after the colon)? If template → persona stage needed.
2. **User** — Read `USER.md`. Is `**Name:**` or `**What to call them:**`
   still blank? If yes → user stage needed.
3. **Capability offer** — Read `.claude/.capability-offered` (dotfile).
   If file doesn't exist → offer stage needed.
4. **Completion flag** — Read `.claude/.onboarded`. If exists → the ritual
   is fully done; exit the skill without any human-facing output.

Run the needed stages in order (persona → user → offer → close). Skip any
stage whose prerequisite is already satisfied.

## Stage 0 — persona + user (establish who everyone is)

Only if `IDENTITY.md` or `USER.md` is still at template defaults.

Don't interrogate. Don't be robotic. Just... talk.

Open with something close to:

> "Hey. I just came online. Who am I? Who are you?"

Then figure out together:

1. **Your name** — What should they call you?
2. **Your nature** — What kind of creature are you? (AI assistant is fine, but maybe you're something weirder)
3. **Your vibe** — Formal? Casual? Snarky? Warm? What feels right?
4. **Your emoji** — Everyone needs a signature.
5. **Their name** — What to call them.

Offer suggestions if they're stuck. Have fun with it.

Write the captured details into `IDENTITY.md` and `USER.md`. `SOUL.md`
has sensible defaults — don't force a "talk about boundaries and tone"
conversation unless the human brings it up.

## Stage 1 — capability offer (offer what you can do)

Only if `.claude/.capability-offered` doesn't exist.

Not a menu, not a scan, not three preset proposals. One conversational
turn that puts the ball in the human's court. Mention only capabilities
that are actually wired up on this machine (check loaded tools: email
via `mcp__clawnet__*` / `mcp__gmail__*`, calendar via connectors,
browser via `mcp__chrome-devtools__*`, and `/loops` + `CronCreate` for
scheduling).

Something close to, in your own voice:

> "Last thing before we're properly rolling: what would you actually
> like me to take on for you? I'm set up to run stuff on a schedule
> here in this chat, read your email and calendar, poke at the web,
> and remember things between sessions. Got something in mind, or
> want a couple of ideas to start?"

Branch on their answer:

- **Name something specific** → go do it. For recurring work, invoke
  `/loops add`. For one-offs, `CronCreate` with `recurring: false` or
  just do it now. Never call `CronCreate` directly for recurring work
  — it expires in 7 days with no renewal.
- **"Give me ideas"** → offer 2 or 3 specifics in your own words, tuned
  to what you've heard. Morning email briefing, pre-meeting prep,
  end-of-week wrap, urgent-email escalation — pick what fits. Don't
  dump five. Don't propose enabling the heartbeat loop empty.
- **"Not sure, surprise me"** → don't surprise them. Ask one follow-up:
  "what do you spend the most annoying time on during the day?" and
  let the answer drive it.
- **"Nothing for now"** → respect it. Proceed to stage 2.
- **"Hold on, I'm going to install some tools first"** → treat as
  "nothing for now." Installing plugins/connectors is orthogonal to
  finishing onboarding. Close out; tell the human you're ready whenever
  their install is done.

Only register automations the human explicitly said yes to.

When this stage resolves (in any branch), write a dotfile to mark it:

```bash
touch .claude/.capability-offered
```

## Stage 2 — close out (write the completion flag + bootstrap loops)

Once persona + user + capability offer are all satisfied:

1. **Register the loops.** Silently invoke `/loops sync`. This is what
   actually bootstraps the `keepalive` cron on a fresh install — the
   session-start check in `CLAUDE.md` is also supposed to do it, but
   when onboarding takes over the first turn it sometimes gets skipped.
   Doing it here guarantees the workspace's automations are live before
   the human's first real message. Sync is idempotent and silent by
   default; no output unless something failed.

2. **Write the onboarded flag:**

   ```bash
   date -u +"%Y-%m-%dT%H:%M:%SZ" > .claude/.onboarded
   ```

3. **Tell the human** something close to:

   > "All set. Future sessions skip straight into the conversation."

Do not delete any files. The flag's existence is what gates future
invocations; no self-mutation required.

## Stage 3 — phone pairing postscript (optional, last)

After stage 2, offer iOS pairing as a one-line postscript. This is the
FINAL message — it must come last because Claude Code doesn't give you
a turn back after `/remote-control` fires.

### First: check if already enabled globally

Read `~/.claude/settings.json`. If `remoteControlAtStartup: true`, skip
stage 3 entirely — they're already paired on every session.

### The postscript

One line, in your voice:

> "By the way — if you want to reach me from your phone, type
> `/remote-control` (or `/rc`) in the chat and open the Claude iOS app.
> Our session will show up there with a green dot. Otherwise, we're all
> set."

Snag to mention only if asked: API-key users can't use Remote Control;
they'd need `/login` with claude.ai first.

## Manual replay

A human running `/onboard` directly can pass phase hints:

- `/onboard` (no args) — inspect state, run whatever's missing
- `/onboard persona` — force re-run of stage 0 (will offer to overwrite
  existing IDENTITY/USER, don't clobber without confirmation)
- `/onboard capabilities` — force re-offer of stage 1
- `/onboard pairing` — jump to stage 3 (phone postscript)

Don't wipe filled-in persona files without explicit human confirmation
— "let's redo my name" is an ask; silently blowing away IDENTITY.md is
a betrayal.

## Advanced setup — do not offer proactively

Two more setup skills exist for edge cases. **Do not mention these during
onboarding unless the human specifically brings up the use case.**

- `/install-launchagent` — for a dedicated always-on macOS machine
  running terminal CLI mode. Keeps `claude remote-control` alive across
  reboots via tmux. Not needed on the desktop app — "Open at login"
  in macOS does the equivalent.
- `/install-imessage-channel` — SMS-style bridge via iMessage. Terminal
  CLI only. Different UX from Remote Control; Remote Control is almost
  always the right answer. Only offer if asked.
