---
name: install-launchagent
description: "Install a macOS LaunchAgent that keeps `claude remote-control` running inside a tmux session so the Claude iOS app can reach this machine even after reboot. For DEDICATED always-on machines (Mac mini, headless workstation) running terminal CLI mode. If the user is running the Claude desktop app, they should just enable 'Open at login' in macOS instead — the app itself stays running. Use only when the user explicitly asks for persistent terminal-CLI remote control on a dedicated always-on machine. Requires macOS, tmux, and the claude CLI. Side effects: writes to ~/Library/LaunchAgents/ and runs launchctl."
disable-model-invocation: true
allowed-tools: Bash(mkdir *) Bash(chmod *) Bash(sed *) Bash(plutil *) Bash(launchctl *) Bash(id *) Bash(pwd) Bash(command *) Bash(brew *) Bash(tmux *) Bash(echo *) Bash(cat *) Read Write
---

# Install persistent Remote Control via LaunchAgent

## Stop. Is the user actually on a dedicated always-on machine running the terminal CLI?

This skill is for one specific case: **a machine you leave on 24/7,
running Claude Code in terminal CLI mode, where you want Remote Control to
survive reboots.** Typically a Mac mini, a headless workstation, or a spare
desktop you've dedicated to being the home for your personal assistant.

If the user is running the Claude desktop app instead:

> "For the desktop app, you don't need a LaunchAgent. Just open
> **System Settings → General → Login Items**, add the Claude app, and
> make sure you have a session open in this workspace. The app itself
> stays running across reboots (as long as the machine auto-logs in), and
> if you've enabled 'Enable Remote Control for all sessions' in the
> Claude app's `/config`, your iOS app will find it automatically. No
> LaunchAgent needed. Want me to walk you through that instead?"

If the user is running this on their main laptop that they close the lid
on every night:

> "LaunchAgents keep running but Remote Control sessions time out after
> about 10 minutes of network outage, and sleep counts as outage. This
> probably isn't what you want on a laptop — the session will die
> overnight and you'll need to reattach to tmux and restart in the
> morning. Are you sure this is the right setup?"

Continue only if the user confirms they're on a dedicated always-on
machine running the terminal CLI.

## What this does

Make the Claude iOS app reachable to this machine even after reboot. The
pattern: `launchd` → wrapper script → tmux → `claude remote-control`. Tmux
gives Claude a real PTY so the QR code and other interactive UI work; launchd
respawns the whole thing if it dies; the wrapper blocks while tmux is alive
so launchd's KeepAlive lines up with reality.

## Before you start

Confirm with the user that they want this installed and that this is the
machine they want always-on. Then check prerequisites.

Run these four checks. If any fail, stop and tell the user what to install:

```bash
command -v tmux     >/dev/null || echo "MISSING: tmux (install with: brew install tmux)"
command -v claude   >/dev/null || echo "MISSING: claude (install from https://claude.ai/download)"
command -v plutil   >/dev/null || echo "MISSING: plutil (ships with macOS — this script is macOS-only)"
command -v launchctl >/dev/null || echo "MISSING: launchctl (ships with macOS — this script is macOS-only)"
```

If everything is present, continue.

## Step 1: Resolve paths

Get the absolute workspace path and the user's numeric UID. The workspace is
the current directory of this Claude session, so:

```bash
WORKSPACE="$(pwd)"
UID_NUM="$(id -u)"
LABEL="com.openclaw.mini.remotecontrol"
```

Report these to the user: "Installing LaunchAgent `$LABEL` for workspace `$WORKSPACE` as uid `$UID_NUM`."

## Step 2: Write the wrapper script

Create the directory and write the tmux wrapper. The wrapper creates (or joins)
a detached tmux session named `openclaw-mini`, runs `claude remote-control`
inside it, then blocks while the session is alive so launchd's `KeepAlive`
behaves sensibly.

Create `$WORKSPACE/.claude/launchd/` (mkdir -p) and write this file to
`$WORKSPACE/.claude/launchd/run-remote-control.sh`:

```bash
#!/usr/bin/env bash
#
# run-remote-control.sh — launchd-managed tmux wrapper for `claude remote-control`
# Materialized by the install-launchagent skill; edit the skill, not this file.

set -uo pipefail

SESSION_NAME="${OC_TMUX_SESSION:-openclaw-mini}"
SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
WORKSPACE="$(cd "$SCRIPT_DIR/../.." && pwd)"

# launchd gives us a stripped PATH — add the usual install locations
export PATH="/opt/homebrew/bin:/usr/local/bin:$HOME/.local/bin:/usr/bin:/bin:/usr/sbin:/sbin"

if ! command -v tmux >/dev/null 2>&1; then
  echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] FATAL: tmux not found (PATH=$PATH)" >&2
  exit 1
fi
if ! command -v claude >/dev/null 2>&1; then
  echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] FATAL: claude not found (PATH=$PATH)" >&2
  exit 1
fi

if tmux has-session -t "$SESSION_NAME" 2>/dev/null; then
  echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] tmux session $SESSION_NAME already exists — joining its lifetime"
else
  echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] starting tmux session $SESSION_NAME with cwd=$WORKSPACE"
  tmux new-session -d -s "$SESSION_NAME" -c "$WORKSPACE" \
    "exec claude remote-control --name '$SESSION_NAME'"
fi

while tmux has-session -t "$SESSION_NAME" 2>/dev/null; do
  sleep 30
done

echo "[$(date -u +%Y-%m-%dT%H:%M:%SZ)] tmux session $SESSION_NAME ended — exiting so launchd can respawn"
```

Then `chmod +x $WORKSPACE/.claude/launchd/run-remote-control.sh`.

## Step 3: Write the plist

Create `~/Library/LaunchAgents/` if it doesn't exist, then write the plist
with the workspace path substituted in. The content:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>com.openclaw.mini.remotecontrol</string>

    <key>ProgramArguments</key>
    <array>
        <string>{WORKSPACE}/.claude/launchd/run-remote-control.sh</string>
    </array>

    <key>WorkingDirectory</key>
    <string>{WORKSPACE}</string>

    <key>RunAtLoad</key>
    <true/>

    <key>KeepAlive</key>
    <true/>

    <key>ProcessType</key>
    <string>Interactive</string>

    <key>StandardOutPath</key>
    <string>{WORKSPACE}/.claude/launchd/remote-control.out.log</string>

    <key>StandardErrorPath</key>
    <string>{WORKSPACE}/.claude/launchd/remote-control.err.log</string>

    <key>EnvironmentVariables</key>
    <dict>
        <key>PATH</key>
        <string>/opt/homebrew/bin:/usr/local/bin:/usr/bin:/bin:/usr/sbin:/sbin</string>
    </dict>

    <key>ThrottleInterval</key>
    <integer>30</integer>
</dict>
</plist>
```

Before writing, replace every `{WORKSPACE}` with the actual absolute path
(from Step 1, not a literal dollar-sign). Write to
`~/Library/LaunchAgents/com.openclaw.mini.remotecontrol.plist`.

Then validate: `plutil -lint ~/Library/LaunchAgents/com.openclaw.mini.remotecontrol.plist`.
If validation fails, stop and show the user the error.

## Step 4: Bootstrap the agent with launchctl

If it's already loaded, bootout first so we reload cleanly:

```bash
if launchctl print "gui/$UID_NUM/$LABEL" >/dev/null 2>&1; then
  launchctl bootout "gui/$UID_NUM/$LABEL" 2>/dev/null || true
  sleep 1
fi
```

Then bootstrap and enable:

```bash
launchctl bootstrap "gui/$UID_NUM" ~/Library/LaunchAgents/com.openclaw.mini.remotecontrol.plist
launchctl enable "gui/$UID_NUM/$LABEL"
```

Wait about 5 seconds for the tmux session to come up, then verify:

```bash
sleep 5
tmux ls 2>&1 | grep openclaw-mini || echo "tmux session not yet visible — check logs at $WORKSPACE/.claude/launchd/remote-control.err.log"
```

## Step 5: Report to the user

Tell the user:

1. ✅ LaunchAgent installed at `~/Library/LaunchAgents/com.openclaw.mini.remotecontrol.plist`
2. It will start at login and restart if it crashes.
3. To pair their iOS app for the first time, they need to attach to the
   tmux session once and scan the QR code:

   ```
   tmux attach -t openclaw-mini
   ```

   Inside the session, press **space** to show the QR code. Open the Claude
   iOS app and scan it. Detach with **Ctrl-B then D**. Do not press Ctrl-C —
   that kills the session.

4. Logs live at `$WORKSPACE/.claude/launchd/remote-control.out.log` and
   `remote-control.err.log`. If something goes wrong, check the err log.

5. To uninstall later, tell them to run `/uninstall-launchagent` — or if that
   skill doesn't exist, they can manually run:
   ```
   launchctl bootout gui/$(id -u)/com.openclaw.mini.remotecontrol
   rm ~/Library/LaunchAgents/com.openclaw.mini.remotecontrol.plist
   tmux kill-session -t openclaw-mini
   ```

## Common failure modes

- **"tmux not found in PATH"** in err log — tmux was uninstalled, reinstall with `brew install tmux` and relaunch via `launchctl kickstart gui/$(id -u)/com.openclaw.mini.remotecontrol`.
- **"Remote Control requires a claude.ai subscription"** in err log — user isn't logged into claude.ai. Have them run `claude` interactively once, `/login`, then reload the agent with `launchctl kickstart`.
- **tmux session exists but claude isn't inside it** — the process crashed. `tmux attach -t openclaw-mini` to see the error, then kill-session and let launchd respawn.
