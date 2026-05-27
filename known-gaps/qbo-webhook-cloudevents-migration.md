---
silo: 8
date: 2026-05-15
status: open — deadline today
last_validated: 2026-05-15
---

# QBO webhook payload structure migrating to CloudEvents — deadline 2026-05-15

## What

Intuit announced QBO webhooks are migrating from legacy payload structure to CloudEvents format. Deadline: **May 15, 2026** (today as of silo 8).

Reference article (surfaced from a prior conversation about this same change):
https://medium.com/intuitdev/upcoming-change-to-webhooks-payload-structure-2a87dab642d0

## Risk if not migrated

QBO → RS webhooks will fail silently after the deadline. This affects:
- Payment status notifications back to RS (the exact silent-gap pattern that produced SO 500950's stuck state on a different mechanism)
- Invoice status updates
- Any other data sync from QBO back to RS

The failure mode is silent — no errors thrown locally, just no events arriving.

## What we don't know (from silo 8)

- Whether RS migrated already. Not investigated this session.
- Which RS edge function receives QBO webhooks. Likely something like `qbo-webhook` or similar in `supabase/functions/`, frozen per STABILITY_RULES.md.
- Whether the existing handler is forward-compatible or hard-coded to the legacy structure.

## How to verify

1. Find the QBO webhook receiver in `supabase/functions/`. Check whether it reads CloudEvents-style fields (`type`, `data`, `source`, `time`) or legacy fields (`eventNotifications[]`, `dataChangeEvent`, etc.).
2. Check Supabase edge function logs for the QBO webhook receiver — recent invocations after May 15 will tell us if Intuit is sending the new format and whether the handler is processing it.
3. If the receiver is legacy-only, this is urgent — payment notifications are silently lost.

## Why deferred

Surfaced at end of silo 8. Not investigated this session. Should be top of next session alongside the SO 501718 Customer Memo blocker.
