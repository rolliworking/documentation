---
silo: 8
date: 2026-05-15
status: open — pending Part 2 button verification
last_validated: 2026-05-15
---

# SO 55153 and 55309 — Stuck-SO Part 2 verification candidates

## Identifiers

| so_number | qbo_invoice_id | order_date |
|---|---|---|
| 55153 | 62451 | 2026-01-22 |
| 55309 | 62599 | 2026-01-29 |

## Source query

```sql
SELECT so_number, order_date, qbo_invoice_id
FROM sales_orders
WHERE qbo_invoice_id IS NOT NULL
  AND is_paid = false
  AND order_date >= now() - interval '120 days'
  AND order_date < now() - interval '21 days'
ORDER BY order_date ASC;
```

These are SOs that:
- Have a QBO invoice attached
- Are unpaid in our DB
- Are within the Ship Station queue's 120-day window
- Are old enough that the cron's 21-row catch-up wouldn't reach them

## Why they're a fixture

These are the next two candidates for the Part 2 button fix to either confirm or contradict. After Part 2 shipped silo 8, a "Sync QBO" click in the Paid Unshipped queue should:
- Call `qbo-invoice-status` against invoice 62451 (for SO 55153)
- Call `qbo-invoice-status` against invoice 62599 (for SO 55309)

Outcomes:
- If QBO returns `isPaid=true` for either: local row catches up, appears in queue as Ready. Confirms Part 2 works and that these were silent-gap SOs like 500950.
- If QBO returns `isPaid=false`: nothing changes locally. Confirms Part 2 works (filter correctly only updated rows that QBO says are paid) but these SOs were legitimately unpaid all along.

Either result is informative.

## What to NOT do

**Don't preemptively UPDATE these rows.** Unlike SO 500950, we haven't confirmed via QBO that they're paid. The investigation report for SO 500950 had an explicit live `qbo-invoice-status` call confirming paid; these don't. Let Part 2's button do its job.

## Recommended next action

Open `/sales/ship-station`, click "Sync QBO" once, watch the network tab for two `qbo-invoice-status` calls against 62451 and 62599. Record the response for each. If either flips, capture the resulting DB state for the audit trail.
