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
