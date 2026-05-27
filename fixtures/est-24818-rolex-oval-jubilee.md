---
silo: 9
date: 2026-05-25
status: ACTIVE — canonical regression test for receive-watch pre-fill on Rolex band jobs
last_validated: 2026-05-25
---

# Fixture: EST 24818 — Rolex Oval Jubilee Bracelet (Original Canonical Worked Example)

## What it is

The estimate that became the canonical "trickle-down architecture works" worked example. Used to verify the receive-watch race condition fix during silo 9.

## Data shape

**Customer:** Richard Wegman (rwegman@aol.com, 8175847033)

**Estimate:**
- estimate_number: 24818
- Line 1: `[B] 20MM Rolex Jubilee Oval SS-`
- Line 2: `BR-OVAL-JUB-ALL` — `Repair - Oval Jubilee...`
- Plus shipping and SVC-CUSTOM lines

**Pre-existing drop-off:**
- Received May 21 (band only)
- "Job profile loaded — BAND ONLY FLOW — Previously received: received:Bracelet on May 21"

## Linked parts row

```sql
SELECT * FROM parts WHERE part_number = 'BR-OVAL-JUB-ALL';
```

| Field | Value |
|---|---|
| part_number | BR-OVAL-JUB-ALL |
| brand | Rolex |
| bracelet_model | Oval Jubilee |
| department | B |
| item_type | service |

## Expected behavior on receive-watch

After silo 9 race condition fix:
- Brand pill: **Rolex** highlighted ✓
- Model field: shows "Oval Jubilee" ✓
- Item Type: **Band Only** ✓
- Department: **B** ✓
- Items Received: Bracelet pill auto-selected

## Verified working

Worked correctly during silo 9 testing on localhost (Cursor + Vite). Confirmed Brand=Rolex, Model="Oval Jubilee" pre-fill on band-only branch.

This is the first time the canonical trickle-down example worked end-to-end across silos.

## Race condition fix evidence

The fix at `src/pages/intake/ClientWatchEntryPage.tsx` lines 177-189 added `hasPrefilledRef.current === formData.estimateId` guard to skip the itemType-change clobber effect when pre-fill just ran for this estimate.

Without the guard: pre-fill set brand=Rolex → jobProfile resolved → setItemType('band_only') fired → effect 177 cleared brand back to '' → Brand pill blank.

With the guard: same sequence, but effect 177 sees ref equals estimate ID, returns early, brand stays Rolex.

## Why this worked but beacon 25530 didn't

EST 24818 brand="Rolex" matches the Brand pill literal `'Rolex'` exactly. Pill highlights correctly even if other downstream bugs exist.

Beacon EST 25530 also has brand="Rolex" (after editing) but still renders blank — proving there's a separate bug not exposed by 24818's data shape. Either:
- itemType sync timing differs by some condition
- Race window only opens under certain estimate shapes
- Supabase embed shape (`li.part` vs `li.parts`) differs by some condition

This is being investigated in silo 10.

## Use this fixture for

- Regression test after any receive-watch pre-fill changes — must continue working
- Reference case for "what should band-only pre-fill look like"
- Baseline before testing more complex flows
