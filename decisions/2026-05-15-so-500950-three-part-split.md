---
silo: 8
date: 2026-05-15
status: Parts 1+2 shipped; Part 3 deferred
last_validated: 2026-05-15
---

# Decision — SO 500950 fix split into Part 1 (data), Part 2 (button), Part 3 (deferred)

## Question

SO 500950 is fully paid in QBO (`balance=0, isPaid=true` per live `qbo-invoice-status` call) but the local `sales_orders` row has `is_paid=false, paid_at=null`. The Ship Station Paid Unshipped queue treats it as not paid → not ready. How to fix?

## Investigation findings

- The `qbo-invoice-sync` cron checks the 21 oldest open invoices per run.
- Invoice 63863 (SO 500950) was paid in QBO but the cron's catch-up never reached it.
- The Ship Station list's manual "Sync QBO" button only re-checked orders created in the last 7 days. SO 500950 is ~44 days old, outside that window.
- Nothing automatic would ever flip `is_paid` for this SO.
- Separately: SO 500950 has `status='fulfilled'` with `tracking_number=null` and all `shipped_qty=0` — something marked it fulfilled without a real shipment. Not the visibility bug, but suspicious.

## Decision: split into 3 parts

### Part 1 (shipped silo 8) — Immediate data unblock

One-row UPDATE: `is_paid=true, paid_at=now()` on `sales_orders` row `a5e827c2-1eb9-4094-b3ba-c79d020d3667`. QBO already confirmed paid, so this just brings local state in sync.

Leaves `status='fulfilled'` alone — separate issue.

### Part 2 (shipped silo 8) — Systemic prevention via button widening

`src/components/sales/PaidUnshippedOrdersList.tsx` lines 350-358 + 414. Replace the 7-day-recent filter with "any unpaid row with `qboInvoiceId` in the visible queue, oldest-first, cap 25 calls per click."

**Why these specific changes:**
- 25-call cap stays in the same order of magnitude as the old 15-cap (rate-limit headroom).
- Oldest-first targets exactly the SOs that the cron's FIFO window misses.
- `unpaid + qboInvoiceId` filter avoids re-calling already-paid rows.
- Touches ONE file only. `qbo-invoice-sync` cron is frozen per STABILITY_RULES.md — Part 2 deliberately doesn't touch it.

### Part 3 (deferred) — Premature-fulfilled source bug

SO 500950's `status='fulfilled'` with `tracking_number=null` and all `shipped_qty=0` and `fulfillment_channel='ship'` shouldn't be possible through legitimate flow. Likely:
- A Pickup Station or fulfill-action path firing on a ship-channel order
- A manual UPDATE somewhere in the codebase
- An edge function side effect

**Why deferred:**
- Doesn't block visibility (the row appears in the queue with isPaid → Ready after Part 1+2).
- Cosmetic until source is found ("fulfilled" label on a row that's not actually shipped).
- Separate investigation; doesn't share root cause with the visibility bug.

## Trade-offs accepted

- Cron remains unfixed (frozen file). 21-row FIFO catch-up will keep missing older paid SOs. Part 2 button mitigates by giving staff a manual catch-all, but it requires a click. If no one clicks, stuck SOs stay stuck.
- Part 3 remains open. Anyone clicking through the queue will see a SO labeled "fulfilled" that hasn't actually shipped. Confusing until investigated.

## What would change this decision

- If multiple silent-gap SOs accumulate before staff notices: raise cron batch size (touches frozen file — needs explicit decision).
- If Part 3 root cause turns out to be a destructive path also affecting active orders: escalate.

## Candidate stuck SOs to verify Part 2

```sql
SELECT so_number, order_date, qbo_invoice_id
FROM sales_orders
WHERE qbo_invoice_id IS NOT NULL
  AND is_paid = false
  AND order_date >= now() - interval '120 days'
  AND order_date < now() - interval '21 days'
ORDER BY order_date ASC;
-- Result silo 8: 55153 (62451, 2026-01-22), 55309 (62599, 2026-01-29)
```

After Part 2, the first "Sync QBO" click should hit both these invoice IDs. If QBO says paid, local row will catch up. If unpaid, no change.
