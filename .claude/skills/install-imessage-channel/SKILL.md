---
name: install-imessage-channel
description: "Install Anthropic's iMessage channel plugin so the user can text this machine from any Apple device and have the messages arrive in a running Claude Code session as a chat bridge. Use ONLY when the user specifically wants SMS-style texting via the iOS Messages app. For normal 'reach my agent from my phone' needs, Remote Control (built-in /remote-control command) is the better answer. Terminal CLI only — the --channels flag is not supported in the Claude desktop app. macOS only. Requires Bun and Full Disk Access."
disable-model-invocation: true
allowed-tools: Bash(command *) Bash(brew *) Bash(claude *) Bash(bun *) Bash(uname *)
---

# Install the iMessage channel plugin

## Stop. Are you sure this is what the user wants?

Remote Control (the built-in `/remote-control` slash command) is almost
certainly what the user wants for phone access. It does a bigger, better
version of what the iMessage channel does: same Claude session, accessible
from the Claude iOS app, full conversation history, live sync between
phone and desktop, works on both terminal CLI and desktop app.

The iMessage channel is only useful if the user specifically wants a
different UX:

- They want to talk to their agent via the **iOS Messages app**, not the
  Claude iOS app. (They'd text their own phone number and get replies as
  iMessages from themselves.)
- They want other people in their iMessage contacts to be able to reach
  the agent by texting, which Remote Control can't do.
- They've tried Remote Control and have a specific reason it doesn't work
  for them.

If none of those apply, stop and tell the user:

> "You probably want `/remote-control` instead. It gives you phone access
> without the iMessage bridge — the Claude iOS app shows this session
> directly, and everything syncs live between phone and desktop. iMessage
> channel is only useful if you specifically want to text me from the
> Messages app, or let other contacts reach me by texting. Which one do
> you want?"

If they confirm they want iMessage channel, continue.

## Surface check: terminal CLI only

The `--channels` flag is a CLI startup argument. The Claude desktop app's
Code tab doesn't support it — there's no equivalent in Settings or
`/config`. You cannot install an iMessage channel for a desktop app
session.

If the user is running the desktop app:

> "This feature only works in the terminal CLI — the desktop app doesn't
> support `--channels`. You have two options:
>
> 1. **Use Remote Control instead** (`/remote-control`). You can keep
>    running the desktop app as your primary interface.
> 2. **Run a parallel terminal session just for the channel.** You'd have
>    the desktop app open for normal work, plus a separate `claude --channels
>    plugin:imessage@...` process running in a terminal for inbound
>    texting. They'd share this workspace's CLAUDE.md, skills, and settings,
>    but be independent sessions.
>
> Which sounds right?"

Continue only if the user picks option 2 or confirms they're in the
terminal CLI.

## Prerequisites

Check the platform:

```bash
uname -s
```

If it's not `Darwin`, stop — iMessage channel is macOS-only.

Check for Bun:

```bash
command -v bun >/dev/null && echo "bun found" || echo "MISSING: bun (install with: brew install oven-sh/bun/bun)"
```

If Bun is missing, have the user install it first.

## Prerequisites

```bash
command -v bun >/dev/null || echo "MISSING: bun (install with: brew install oven-sh/bun/bun)"
```

The plugin is a Bun script. If `bun` is missing, tell the user to install it
first and stop here.

Also confirm we're on macOS. Run `uname -s` — if the output is not `Darwin`,
tell the user the iMessage channel is macOS-only and offer the Telegram or
Discord channels instead.

## Step 1: Install the plugin

Tell the user what's about to happen, get confirmation, then run:

```
/plugin install imessage@claude-plugins-official
```

If Claude Code reports the plugin isn't in any marketplace, run:

```
/plugin marketplace add anthropics/claude-plugins-official
```

then retry the install. After install, run `/reload-plugins`.

## Step 2: Full Disk Access

The iMessage database is protected by macOS. Tell the user:

> "When the channel first runs, macOS will prompt for **Full Disk Access** for
> whatever terminal or app is running Claude Code. Click **Allow**. If you
> missed the prompt or clicked Don't Allow, grant it manually at
> **System Settings → Privacy & Security → Full Disk Access** and add your
> terminal (Terminal, iTerm, Ghostty, or the Claude desktop app) to the list."

Don't try to automate this — it requires a macOS system dialog the user has
to click through.

## Step 3: Restart with `--channels`

The iMessage plugin only receives messages while Claude Code is running with
the `--channels` flag. Tell the user:

> "Exit this Claude session (Ctrl-D or `/exit`), then restart from the same
> directory with:
>
> ```bash
> cd $(pwd)
> claude --channels plugin:imessage@claude-plugins-official
> ```
>
> On the first restart, macOS will pop up the Full Disk Access prompt. Allow
> it, then quit and start Claude once more. From then on, the channel is live
> whenever you're in a session started with that flag."

If the user is running in the Claude desktop app Code tab rather than the
terminal CLI, channels are terminal-only today. Tell them they need to open
a terminal and use the CLI for iMessage. The rest of the workspace (persona,
skills, memory) still works identically in both surfaces — just this one
feature is CLI-exclusive.

## Step 4: Pair by texting yourself

Self-chat bypasses access control. Tell the user:

> "Open Messages on any device signed into your Apple ID and send a message
> to yourself. It'll reach me immediately. The first time I reply, macOS will
> pop up an Automation prompt asking if your terminal can control Messages.
> Click OK."

## Step 5: Allow other senders (optional)

By default, only the user's own messages pass through. To let another contact
reach the agent, the user runs:

```
/imessage:access allow +15551234567
```

or

```
/imessage:access allow user@example.com
```

Tell them about this option but don't set it up unless they ask.

## Report to the user

After the plugin is installed and the user has restarted with `--channels`:

1. ✅ Plugin installed from `claude-plugins-official`.
2. ✅ User has granted Full Disk Access (or will on first run).
3. 📱 To test, text yourself from any Apple device on the same Apple ID.
4. 🔐 To allow other people to reach you, run `/imessage:access allow <phone-or-email>`.
5. ⚠️ This only works while Claude Code is running in a terminal with
   `--channels plugin:imessage@claude-plugins-official`. If they want it
   always-on, they should also install the LaunchAgent via
   `/install-launchagent` — but note that the LaunchAgent wrapper would need
   to be modified to include the `--channels` flag in the claude invocation.
