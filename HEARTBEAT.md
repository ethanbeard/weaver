# HEARTBEAT.md

This file is the checklist the `heartbeat` loop reads every 30 minutes
(if enabled). Add actionable items here to have the agent periodically
check on things without you having to ask.

The loop itself is defined at `.claude/loops/heartbeat.md` and is
**disabled by default**. To turn it on:

```
/loops enable heartbeat
```

To turn it off:

```
/loops disable heartbeat
```

If this file is empty or only contains comments/headers, heartbeat fires
stay silent — no chat clutter. The agent reads the file each fire, so
add or remove items any time and changes take effect on the next fire.

The loop is session-scoped: it fires in the currently active session
and its output appears inline in this chat (not in a separate sidebar).
The `keepalive` loop handles renewal past the 7-day cron expiry
automatically.

## Items

<!-- Add actionable items below. Examples:

## Inbox
- Check email for anything urgent from contacts on the watchlist.
  Summarize in one line if found; stay silent otherwise.

## Calendar
- If a meeting starts in < 15 min and the human hasn't been pinged
  about it yet, remind them with a one-line heads-up.

## PRs
- Glance at open pull requests; flag any with failing CI or unanswered
  review comments from the last hour.

-->
