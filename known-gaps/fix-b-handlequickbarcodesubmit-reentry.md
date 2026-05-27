---
silo: 7
date: 2026-05-14
status: scoped, prompt drafted, not applied
last_validated: 2026-05-14
---

# Fix B — handleQuickBarcodeSubmit re-entry guard

## The bug

`handleQuickBarcodeSubmit` in `src/pages/sales/SalesOrderDetailPage.tsx` (lines 525-704) is async, performs ~600ms of Supabase round-trips, and has no re-entry guard. It sets `isQuickSearching = true` on line 529 to disable the submit button, but the `<form onSubmit>` still fires on Enter regardless of button state. A second Enter press during the async window starts a parallel handler invocation with the same `quickBarcode` value (which isn't cleared until line 689 after the await chain finishes). Both invocations find the same part and append `[..., newItem, blankLine]` to lineItems.

Result: same part appended twice in one save. Forensic evidence on SO 501694 — `2130-540` inserted twice in same batch at sort_orders 9 and 11; session replay showed user entered Part 206C, Recut Fluted bezel, and Bezel Laser Welding twice each in succession.

## The fix

One line: `if (isQuickSearching) return;` at top of handler, after `e.preventDefault()`, before `setIsQuickSearching(true)`.

## Scope

- Touch ONE function: `handleQuickBarcodeSubmit` in `src/pages/sales/SalesOrderDetailPage.tsx`.
- Out of scope: addLineItem, persistOrder, draft restore, PartsSearchInput, autocomplete, picker, button disabled state.

## Status

Scoped silo 7. Prompt drafted at end of session. User pivoted to silo packaging before pasting to Lovable. Should ship next session.

Independent of Fix A (already applied to test branch). Both fixes ship to the same file but address different defects:
- Fix A stops Write 3 (post-navigation re-insert)
- Fix B stops Write 2 in-batch dup (same part inserted twice in one save)
