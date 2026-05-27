---
silo: 7
date: 2026-05-14
status: tech debt, not blocking
last_validated: 2026-05-14
---

# Shop floor commit asymmetry — commitDrop vs handleCommit

## The gap

`src/pages/ShopFloor.tsx` has TWO commit handlers performing similar updates to `job_components` and `job_component_logs`:

1. **`commitDrop`** (lines 645-689) — drag-drop + TechPickerDialog flow
2. **`handleCommit`** (lines 995-1060) — scan-to-station queue flow (barcode scanner panel)

Both update `job_components` and insert into `job_component_logs`. The code is similar but NOT identical:
- Different update object shapes
- Different log insert field sets
- `handleCommit` writes `assigned_to: tech?.id`; `commitDrop` hardcodes `null`
- `commitDrop` uses `action: "status_change"`; `handleCommit` uses `action: isLock ? "in_safe" : "assigned"`

## Why it matters

Any change that affects "what gets written when a dot moves" requires patching BOTH handlers. The silo 7 B1 attempt patched only `handleCommit`, which broke the drag flow. Required B2 follow-up patch in the same session.

This is a real footgun for future shop floor changes.

## Suggested fix

Extract a shared helper (e.g. `commitComponentMove({ componentId, jobId, station, tech, ... })`) that handles both the component update and the log insert. Both handlers call it. Differences in `action` value or other handler-specific details get parameterized in.

## Status

Not blocking — both handlers work after B1+B2. But the asymmetry will bite again on the next change that touches both paths. Worth addressing before more shop floor work lands.
