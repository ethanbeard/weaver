---
name: loops
description: "Manage in-chat scheduled prompts. Each file in .claude/loops/ is a loop: YAML frontmatter (cron, enabled, description) + prompt body. Subcommands: list (default), add, remove, enable, disable, sync. Sync reconciles the registry with CronList — registers missing loops, renews near-expiry ones, cancels orphans. Loops fire as new turns in the active session; output appears inline in the current chat. A daily keepalive loop re-runs /loops sync to keep everything alive past the 7-day session-scoped cron expiry. Run /loops sync silently at session start (via AGENTS.md)."
argument-hint: "[list|add|remove|enable|disable|sync] [<name>]"
allowed-tools: Read Write Bash(ls *) Bash(rm *) Bash(date *) Bash(cat *) Glob
---

# Loops — manage in-chat scheduled prompts

Each file in `.claude/loops/` is a **loop**: a session-scoped scheduled
prompt that fires as a new turn in the currently active session. Output
appears inline in the chat (synced to the Claude iOS app via Remote
Control). Not in a sidebar, not via email, not via notification.

This skill is the only way loops should be registered and renewed. Don't
call `CronCreate` directly for recurring workspace automations — add a
loop file and let `sync` wire it up.

## Loop file format

`.claude/loops/<name>.md`:

```markdown
---
cron: "<5-field cron expression>"
enabled: true
durable: false
description: <one-line description>
---

<the prompt the agent fires with each time this loop runs>
```

The filename (minus `.md`) is the loop's name. `cron` accepts standard
5-field cron expressions. `enabled: true` means `sync` should register
it; `false` means skip.

Optional frontmatter fields:

- `durable: true` — hint to the harness to persist the cron to
  `.claude/scheduled_tasks.json`. Currently gated behind a feature flag
  in Claude Code (`tengu_kairos_cron_durable`) and silently downgraded
  to session-only in most installs. Setting it doesn't hurt and will
  auto-upgrade behavior when/if the flag rolls out. Only `keepalive`
  ships with it by default.
- `debounce_seconds: <n>` — window in seconds during which a duplicate
  fire is suppressed. Defaults to `60`. See the multi-session dedup
  section below.
- `last_fired: <ISO UTC timestamp>` — agent-managed. Written by the
  loop's own preamble on each successful fire; read by subsequent fires
  to suppress duplicates. Don't edit this by hand.

## Multi-session dedup

Claude Code's crons are session-scoped. If two sessions are open against
the same workspace, each will register its own copy of every loop and
fire them independently — you'd see a morning briefing twice.

The dedup mechanism: every registered loop's prompt starts with a
preamble that reads the loop's own md file, checks `last_fired`, and
bails silently if it's within `debounce_seconds` of now. Only the first
session to win the race writes `last_fired` and proceeds with the work;
the rest see the fresh timestamp and end their turn silently.

This is a best-effort debounce, not a hard lock. A race window exists
if two sessions read-then-write within the same ~100ms. For typical
fire intervals (30+ minutes) the race is harmless.

## Argument parsing

Read `$ARGUMENTS`. First token is the subcommand:

| Subcommand (default: `list`) | Action |
|---|---|
| `list` or empty | Show all loops with status |
| `add` | Interactive flow to create a new loop file |
| `remove <name>` | Delete the loop file and cancel its cron |
| `enable <name>` / `disable <name>` | Toggle `enabled` in the file's frontmatter |
| `sync` | Reconcile CronList with the registry (the main reconciliation op) |

If `$ARGUMENTS` is anything else, ask the human what they wanted.

---

## How sync works

`sync` is the heart of this skill. It's idempotent — running it twice in
a row should be a no-op the second time (if nothing changed on disk).

### Step 1: Bootstrap the keepalive

The `keepalive.md` loop is what runs `/loops sync` daily so other loops
don't expire. If it's missing, recreate it with default content:

```markdown
---
cron: "7 3 * * *"
enabled: true
durable: true
description: Daily renewal — runs /loops sync to keep all loops alive past the 7-day cron expiry
---

Silently run `/loops sync`. If any loops are within 48 hours of expiring,
renew them. If new loop files have been added since the last sync, register
them. If loop files have been removed or disabled, cancel their matching
scheduled tasks. Produce no output unless something fails — this is
plumbing, not user-facing.
```

If `keepalive.md` exists but is disabled, **do not** force-enable it. The
human may have deliberately disabled it. Warn instead: *"keepalive is
disabled; without it, loops will die after 7 days unless you manually
re-sync."*

### Step 2: Read the registry

`ls .claude/loops/*.md` (or `Glob`), then `Read` each. For each file, parse
the YAML frontmatter (cron, enabled, description) and the body (prompt).

Build a map: `{name → {cron, enabled, prompt}}`.

### Step 3: Inspect CronList

Call `CronList`. It returns each registered cron's ID, cron expression,
prompt, and recurring state.

Filter to loop-managed entries: their prompts start with the marker line
`<loop:NAME:ISO_TIMESTAMP>`. Parse each marker to extract the loop name
and the creation timestamp.

Build a map: `{name → {id, cron, registered_prompt_body, created_at}}`.

Non-loop-managed entries (no `<loop:...>` marker) are left alone — they
might be `/loop` runs the human created manually, or other agent-created
tasks.

### Step 4: Reconcile

For each name in the union of registry and CronList:

| Registry | CronList | Action |
|---|---|---|
| enabled | absent | `CronCreate` with marker prefix |
| enabled | present, same cron+prompt, fresh (< 5 days old) | leave alone |
| enabled | present, different cron OR prompt body | `CronDelete` old, `CronCreate` new |
| enabled | present, old (>= 5 days old) | `CronDelete` old, `CronCreate` new (renew before 7-day expiry) |
| disabled | present | `CronDelete` (respect the disable) |
| disabled | absent | leave alone |
| absent (no file) | present | `CronDelete` (orphan cleanup) |
| absent | absent | leave alone |

### Step 5: Register loops

For each loop to register, build the prompt with a dedup preamble + the
file body:

```
<loop:NAME:TIMESTAMP>
Multi-session dedup — do this first, before anything else:

1. Read `.claude/loops/NAME.md`.
2. Look at `last_fired` in the frontmatter. If it exists and is within
   the last DEBOUNCE seconds of now (DEBOUNCE = the `debounce_seconds`
   frontmatter field, default 60), end this turn silently. Another
   session already claimed this fire — producing any output would be
   duplicate noise.
3. Otherwise, update the frontmatter: set `last_fired` to the current
   ISO 8601 UTC timestamp (e.g., `last_fired: 2026-04-21T17:30:00Z`).
   Use `date -u +%Y-%m-%dT%H:%M:%SZ` to generate it. Write the file
   back before proceeding.

Then execute:

<body from the file>
```

Where `NAME` is the filename minus `.md`, `TIMESTAMP` is the current
moment in ISO 8601 UTC (e.g., `2026-04-16T10:30:00Z`) via
`date -u +%Y-%m-%dT%H:%M:%SZ`, and `DEBOUNCE` is the literal value from
the loop's `debounce_seconds` frontmatter (or `60` if not set).

Then call `CronCreate(cron=<from frontmatter>, recurring=true, durable=<from frontmatter, default false>, prompt=<assembled>)`.

**Why the dedup preamble:** Claude Code's crons are session-scoped, so
multiple sessions in the same workspace each register their own copy
and fire independently. The preamble makes every fire read-check-write
the loop's own md file — only the first session to win the race
proceeds; the rest see the fresh `last_fired` and bail. This is
best-effort (small race window), not a hard lock.

**Why `durable`** is passed: it's a hint to the harness for restart
survival. Currently ignored in most installs (gated behind a
GrowthBook flag) but costs nothing to include — auto-upgrades behavior
if the flag ever ships. Only `keepalive` is durable by default.

### Step 6: Report (silent by default)

If sync was called by the human (`/loops sync` explicitly) → summarize
what happened: "Registered 2 loops, renewed 1, cancelled 0 orphans."

If sync was called by the keepalive or by `AGENTS.md` startup → stay
silent unless something failed. No chat spam.

---

## List (default subcommand)

Show the human what's registered.

1. Read all `.claude/loops/*.md` files.
2. Call `CronList`, filter to loop-managed entries.
3. **Self-heal if needed.** If `keepalive.md` has `enabled: true` but
   no matching cron is registered, the session-start sync was skipped.
   Run `sync` automatically before rendering the table — don't just
   flag the discrepancy to the human. They asked "what's running," not
   "what's broken." Fix it, then show the result.
4. Render as a table:

```
Loop            cron              enabled   registered   next fire       age
─────────────── ───────────────── ───────── ──────────── ─────────────── ──────
keepalive       7 3 * * *         yes       yes          tomorrow 3:07a  2d
heartbeat       */30 * * * *      no        no           —               —
morning         3 7 * * 1-5       yes       yes          tomorrow 7:03a  4d
```

Include a one-line footer with any issues:

- `heartbeat.md has enabled: true but no cron registered — run /loops sync`
- `keepalive.md is disabled — loops will die after 7 days`

---

## Add

Interactive. Ask the human for:

1. **Name** — turns into `<name>.md` (kebab-case, no spaces).
2. **Cron** — 5-field expression. Offer common patterns: every 30 min
   (`*/30 * * * *`), weekday 8am (`3 8 * * 1-5`), Monday 9am (`0 9 * * 1`).
3. **Description** — one line for the frontmatter.
4. **Prompt** — what the agent should do each fire. Remind them:
   - Keep it tight. This prompt runs every fire.
   - Include a silence rule: *"If nothing actionable, stay silent."*
   - Reference absolute paths or workspace-relative paths the agent can
     resolve.

Write `.claude/loops/<name>.md` with the assembled frontmatter + body.
Then run `sync` to register it immediately.

---

## Remove

`remove <name>`:

1. Confirm with the human — removal is destructive.
2. `rm .claude/loops/<name>.md`.
3. Run `sync` — the orphan-cleanup step will cancel the cron.

If `<name>` is `keepalive`, **stop** and warn: "The keepalive is what keeps
the other loops alive past the 7-day expiry. Removing it will break your
other loops unless you manually sync regularly. Are you sure?" Confirm
before proceeding.

---

## Enable / Disable

`enable <name>` / `disable <name>`:

1. Read `.claude/loops/<name>.md`.
2. Edit the frontmatter: set `enabled: true` or `enabled: false`.
3. Write back.
4. Run `sync` — the next reconciliation will register or cancel as needed.

---

## Notes

- **Loops are session-scoped.** If the Claude Code session dies, the
  registered crons die too. `--resume` restores them if they haven't
  expired. Past 7 days, everything is gone and the session-start sync
  in `AGENTS.md` rebuilds the state from the registry. `durable: true`
  is supposed to persist crons to `.claude/scheduled_tasks.json` but is
  currently gated and typically silently downgraded to session-scoped;
  don't rely on it for correctness.
- **Multi-session coexistence.** Multiple Claude Code sessions against
  the same workspace is supported. Each session independently registers
  its copy of every loop, but the dedup preamble in each registered
  prompt ensures only one session actually does the work per fire. If
  you want a loop to NEVER dedupe (i.e., fire once per session on
  purpose), set `debounce_seconds: 0` in its frontmatter.
- **The keepalive is special but not special-cased.** It's just another
  loop file. The only bit of special logic is bootstrapping it if missing
  (step 1 above) — because without it, the whole system can't self-heal.
- **Marker format matters.** `<loop:NAME:TIMESTAMP>` must be on the very
  first line of the registered prompt, followed by a newline, so parsing
  is unambiguous. The body starts on line 2.
- **Don't register non-loop scheduled tasks via this skill.** Things that
  shouldn't auto-renew or that are one-off should use `CronCreate`
  directly without the `<loop:...>` marker. `/loops sync` ignores
  unmarked cron entries.
