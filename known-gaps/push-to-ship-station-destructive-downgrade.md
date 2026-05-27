---
silo: 8
date: 2026-05-15
status: open — no confirmation dialog on destructive action
last_validated: 2026-05-15
---

# "Push to Ship Station" downgrade branch wipes shipment evidence with no confirmation

## What

`SalesOrderDetailPage.tsx` lines 2069-2093, the "Push to Ship Station" handler:

- Always sets `fulfillment_channel = 'ship'`
- Only on downgrade from `status='shipped'`:
  - `status = 'fulfilled'`
  - `tracking_number = null`
  - `ship_date = null`
- Plus `so_lines` UPDATE: `shipped_qty = 0` for all lines on that SO

There is no confirmation dialog before the wipe. Any user who can save the SO can click Push and silently destroy shipment evidence on a shipped order.

## Risk

If a staff member misclicks Push on a row whose status is `shipped` (intending to push a sibling or a different order, or just navigating fast), they will wipe:
- The tracking number
- The ship date
- All line-level shipped quantities

These are not recoverable from local state — only via QBO or ShipStation-external records, if any exist.

## Why this exists

Investigation found no documented reason. The branch reads as "let user re-push a shipped order back to the ship queue," but it doesn't preserve the prior shipped state anywhere first. Likely a pattern that made sense at some earlier point and never got revisited.

## What would fix it

Either:
- Add an `AlertDialog` confirmation when `status === 'shipped'`: "This will clear the tracking number, ship date, and reset all line shipped quantities. Are you sure?"
- Or: when downgrading from `shipped`, snapshot the prior state to an audit table before clearing.

Confirmation is simpler. Snapshot is safer. Either is better than current.

## Why deferred

Not actively biting today. Surfaced silo 8 during "is Push to Ship Station an override?" investigation. Worth knowing before someone hits it; not urgent enough to ship same session.
