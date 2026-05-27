---
silo: 9
date: 2026-05-25
status: OPEN AT CHAT CLOSE — active debug, three bug theories not yet conclusively resolved
last_validated: 2026-05-25
---

# Gap: Beacon EST 25530 Still Renders Blank Brand/Model on Receive-Watch

## Symptom at chat close

EST 25530 receive-watch on localhost:8080 renders:
- Item Type: **Band Only** ✓ (correct)
- Department: **B (Band Room)** ✓ (correct)
- "Previous Watches" pill: **(BR-Tracking-Beacon)** ✓ (part identified)
- Brand pill: **all three blank** (Rolex/Tudor/Other none highlighted) ❌
- Size pill: blank ❌
- Material pill: blank ❌
- Model field: blank (just placeholder) ❌

Despite SQL confirming:
- `parts.brand = 'Rolex'` for BR-Tracking-Beacon
- `parts.bracelet_model = 'Tracking-Beacon-Model'`
- `parts.department = 'B'`
- `parts.item_type = 'service'`
- Estimate line 2 `part_id` correctly links to that parts row

## Three theories investigated, none resolved

### Theory 1: HMR/stale bundle (Cursor RULED OUT)
Cursor curl'd `http://localhost:8080/src/pages/intake/ClientWatchEntryPage.tsx` and confirmed the new matcher code is in the served bundle.

### Theory 2: sortedLines drops line 2 (Cursor RULED OUT by code read)
sortedLines construction at line 834-836 has no filter, only sort by sort_order. SQL confirms 2 lines in estimate_line_items.

### Theory 3: Downstream consumption issue (Cursor CONFIRMED via code analysis, FIXES UNVERIFIED in browser)

Three sub-bugs identified:

**Bug A — Brand pill literal-match**
Pills only highlight when `formData.brand === 'Rolex' | 'Tudor' | 'Other'` literally. Curated brand "Tracking-Beacon-Brand" (earlier beacon version) didn't match — RESOLVED by changing beacon brand to "Rolex".

**Bug B — itemType not synced in pre-fill**
Pre-fill writes to `formData.bracelet_description` but itemType state might still be 'watch' (waits for jobProfile effect). Form renders watch layout reading from `formData.model` (empty). PROPOSED FIX: setItemType('band_only') inside pre-fill when isBandOnly true.

**Bug C — clobber guard race**
Effect 177 guard uses `hasPrefilledRef.current === formData.estimateId`. But `formData.estimateId` is async state; there's a render where ref is set but state isn't. Effect 177 fires, clears brand. PROPOSED FIX: drop the estimateId comparison, just check `hasPrefilledRef.current` truthiness.

Cursor applied Fix 2 + Fix 3 (Bug B and Bug C resolutions). Local testing on EST 25530 still showed blank Brand pill. Either:
- Fixes saved to disk but Vite HMR didn't pick up (Cursor's curl verification was earlier in the session, may have gone stale)
- Fixes correct but bug deeper than caught (Supabase embed shape — `li.part` object vs `li.parts` array — never explicitly verified at runtime)
- Race condition we still haven't isolated

## What's NOT yet been done

- **Runtime console.log inspection of `sortedLines[1]` shape** — would conclusively show whether `.part` is object or `.parts` is array
- **Bulldozer prompt sent to Cursor** with raw-shape inspection at every DB hop, runtime log capture, intake payload capture. Drafted but not sent before chat hit limit.

## What worked for EST 24818 but didn't generalize

EST 24818 (Rolex Oval Jubilee, BR-OVAL-JUB-ALL) pre-fills correctly with Brand=Rolex, Model="Oval Jubilee". Initial theory was "the architecture works." Beacon EST 25530 with explicit Rolex brand proved it doesn't — 24818 may have worked by accident (race condition not triggered on that specific data shape, or itemType happened to be already 'band_only' before pre-fill).

## Next step for silo 10

Send bulldozer prompt drafted at chat close. Includes:
1. Per-step raw JSON of DB results (no paraphrasing)
2. Runtime `[BEACON TRACE]` console.log capturing `sortedLines[1]?.part` vs `sortedLines[1]?.parts?.[0]` shapes
3. Intake payload capture (caret string contents before fetch)
4. Continue past every break with mocked data — get full break map in one pass

## Evidence

- SQL Editor result during silo 9 confirming parts row + estimate join data correctness
- Cursor's investigation report ("Cause 1/2/3" analysis) identifying Bug A/B/C
- Multiple screenshots of EST 25530 receive-watch showing blank fields
- Beacon part rows curated in `/inventory/parts`:
  - BR-Tracking-Beacon: brand=Rolex, bracelet_model=Tracking-Beacon-Model
  - BR-Tracking-Beacon2: brand=Rolex, bracelet_model=Tracking-Beacon-Model2
