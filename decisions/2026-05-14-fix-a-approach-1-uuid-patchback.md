---
silo: 7
date: 2026-05-14
status: applied to test branch, test 1 passed, test 2 pending
last_validated: 2026-05-14
---

# Fix A approach — UUID patch-back over guard widening

## Context

The SO duplicate-line bug (Bug 2 in silo 7 silo doc) had two viable approaches:

**Approach 1: Patch local state with assigned UUIDs after INSERT.**
After `persistOrder`'s batch insert succeeds, use `.insert(...).select()` to capture assigned UUIDs, then `setLineItems` to patch local rows by position so subsequent saves match `existingLineMap`. Invisible to user.

**Approach 2: Widen the asymmetric guard.**
Change the guard at `SalesOrderDetailPage.tsx:784-796` from "UUID-shaped local id without DB match → throw" to "any local id without DB match → throw." Simpler diff. But user can't save until they refresh — every navigation back becomes a "refresh required" error instead of a silent duplicate.

## Decision

**Approach 1.** User chose explicitly via in-chat option picker.

## Reasoning

1. **Root-cause fix vs symptom-surfacing.** Approach 1 fixes WHY duplicates happen (local state never gets DB UUID). Approach 2 just surfaces the failure as a user-visible error.
2. **No new UX friction.** Approach 2 would block saves until refresh, which workers would experience as a regression even though it's preventing a bug.
3. **Autosave race covered.** After patch-back, autosave writes the now-UUID-keyed lineItems to localStorage draft. Restore on remount overlays correct UUIDs onto fresh DB hydration — they match — no spurious INSERT.

## What would change the decision

- If patch-back has unexpected interaction with autosave or draft (e.g. autosave captures mid-hydration state with mixed UUID/non-UUID ids).
- If concurrent edits cause the patch-back to overwrite a user's recent changes.
- If the `.select()` on batch INSERT returns rows in different order than the input array (would require matching by `sort_order` instead of position).

None of these issues observed in Test 1 (save → save → refresh on test branch: passed). Test 2 (with navigation, the production failure mode) pending.

## Files affected

- `src/pages/sales/SalesOrderDetailPage.tsx` — `persistOrder` INSERT branch

## Out of scope

- Asymmetric guard at lines 784-796 stays. Now unreachable from normal-save path but still valuable as defense-in-depth.
- `addLineItem` (line ~431) and `handleQuickBarcodeSubmit` (line ~525) continue producing `String(Date.now())` ids. Fix is at persistOrder, not the add side.
- Draft restore logic at lines 175-210 unchanged. The patch-back ensures the draft contains correct UUIDs going forward.
