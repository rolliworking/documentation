---
silo: 7
date: 2026-05-14
status: known schema choice, not addressed
last_validated: 2026-05-14
---

# Performance Report — no unique constraint on `sales_order_id`

## The gap

`performance_verification_reports.sales_order_id` is a nullable FK with NO unique constraint. The schema physically allows multiple reports per SO.

The lookup hook `usePerformanceReportBySalesOrder` (`src/hooks/usePerformanceReport.ts:51-66`) handles multiple by returning the most recent via `order created_at desc limit 1`.

## Why it matters

If a user ever creates two reports for one SO (intentionally or via a double-click on the dropdown or a navigation race), only the most recent is reachable from the SO. The older one becomes orphaned — no list page exists, no link from any other page. It sits in the DB invisibly.

No production incidents observed yet.

## Possible fixes

1. **Add unique constraint** `UNIQUE (sales_order_id) WHERE sales_order_id IS NOT NULL`. Prevents the orphan condition entirely. Backfill audit required to confirm no SOs currently have multiple reports.
2. **Add list page** showing all reports for a SO, so orphans aren't invisible.
3. **Both.**

## Status

Identified during silo 7 investigation. Not addressed. Worth a quick query before deciding: `SELECT sales_order_id, COUNT(*) FROM performance_verification_reports WHERE sales_order_id IS NOT NULL GROUP BY sales_order_id HAVING COUNT(*) > 1`.
