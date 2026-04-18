---
cron: "*/30 * * * *"
enabled: false
description: Periodic HEARTBEAT.md checklist sweep
---

Read `HEARTBEAT.md` in the workspace root.

- If it's empty, or contains only comments/headers/empty list stubs:
  **end the turn immediately with zero output.** Do not type `HEARTBEAT_OK`,
  do not type "nothing to do", do not acknowledge the fire. Silence is the
  correct response.
- Otherwise, follow the actionable items listed there. One concise update
  per fire. Prefer silence over filler — only produce output when
  something genuinely warrants the human's attention.
