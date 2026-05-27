---
silo: 7
date: 2026-05-14
status: trigger anchor, root cause traced to different SO
last_validated: 2026-05-14
---

# EST-12054 / SO 501690 — Lee Mitchell "different versions" complaint

## What it is

An estimate / sales order in RolliSuite that Mike used to report what appeared to be "two different versions of the same SO" — opening from estimate's "View SO" button vs. opening from search showed different totals.

## Identifying data

- **System:** RolliSuite
- **Estimate number:** E12054
- **SO number:** 501690
- **Customer:** Lee Mitchell (herbert.lee.mitchell@gmail.com, 704-840-3418)
- **Estimate date:** 10/05/2025
- **Line item:** W-SERV-VINT-1 (Rolex Level 1 Vintage Basic Service, $750)

## Why it became a fixture

Mike's initial framing was "the double entry bug might be combining the two different SO#501690 living in different parallel universes." Investigation traced this hypothesis to be:

**Wrong in the way Mike framed it.** There is exactly one `sales_orders` row for 501690. Both "View SO" paths route to the same `/sales/orders/:id` component with the same React Query cache keys.

**Right in spirit.** The "different versions" Mike was seeing came from the localStorage draft layer (`so-draft-${id}`). When `draft.savedAt > order.updated_at`, the draft restore overlays unsaved state on top of fresh DB hydration. Path A (estimate → View SO button) and Path B (search → click result) both land on the same component and same draft restore logic, but the search dropdown shows `sales_orders.total_amount` directly (last-persisted value), while the detail page shows recomputed total from `lineItems` (possibly draft-overridden). Hence "different totals."

This investigation became the entry point for the bigger SO duplicate-line root cause work (see SO 501694 fixture).

## How to use it

Mostly historical now — the canonical test cases are SO 501694 (for the duplicate bugs themselves) and the localStorage draft layer for understanding the architecture. EST-12054 is useful only as the "first reported symptom" anchor for understanding how Mike's framing initially pointed in the wrong direction.

## What this fixture taught

- Mike's initial framing of a bug is data, not gospel. The "two parallel universes" hypothesis was partially right but pointed at the wrong layer (database vs. localStorage).
- The localStorage draft mechanism in `SalesOrderDetailPage.tsx` is load-bearing for multiple bug patterns. Worth understanding fully before designing fixes that touch it.
