---
silo: 9
date: 2026-05-25
status: partially closed — Fix A+B shipped on test, more fields still uncovered
last_validated: 2026-05-25
---

# Gap: RW Receiver Writes Only 4 of 17 `job_components` Columns

## What's missing

`rolliworking/supabase/functions/rollisuite-intake/index.ts` INSERTs `job_components` rows with only `{job_id, component_type, label_id, status}`.

Columns left NULL at INSERT (before silo 9 fixes):
- `department` — fixed in silo 9 via Fix B (commit `3bee3db`)
- `station_id` — still NULL
- `assignee_name`, `assignee_initials`, `assigned_to`, `assigned_watchmaker_id`, `assigned_dept_tech_id` — still NULL
- `notes` — still NULL
- `cancelled_*` columns — still NULL (expected default)

## Why it matters

`/shop-floor` lookup (`src/pages/ShopFloor.tsx` L1306-1325) reads `department, station_id, assignee_name, assignee_initials` from `job_components`. With these NULL after intake, three backfill flows compensate every time:

- `BackfillPanel` in ShopFloor L1481-1674 — manual button when components.length===0
- `backfillWatchmakerJobs` in ComponentStatus.tsx L527-579 + silent useEffect mirror L582-623 (runs on every page mount)
- `BackfillBandJobs.tsx` admin tool

## What's been fixed in silo 9

- **Fix B (commit `3bee3db`)** — Receiver now sets `job_components.department` at INSERT. Helper `componentDeptFor()`:
  - `watch_head` → `watchmaker_bench`
  - `case` → `refinishing`
  - `bracelet` → `band_repair` (or `precious_metals` if PM code present)
- Eliminates the `BackfillPanel` button appearing for new intakes.

## What's still open

- `station_id` left NULL — falls back to derivation in `dotStationFor()` L155-208 of ShopFloor
- `assignee_*` and `assigned_*` columns — never populated on intake (Fix E in backlog, ~20 lines)
- The silent `ComponentStatus` useEffect at L582-623 still fires on every page mount because re-intakes for already-assigned jobs need it

## Resolution path

Fix E in silo 9 backlog: when receiver runs against a job that already has `jobs.assigned_watchmaker`, populate the new `job_components` row with matching `assignee_name`, `assignee_initials`, `assigned_watchmaker_id`. Resolve watchmaker via initials from `watchmakers` table.

## Evidence

- `rolliworking/supabase/functions/rollisuite-intake/index.ts` lines 577-598 — INSERT blocks for the three flows
- `rolliworking/src/pages/ComponentStatus.tsx` L527-579 (manual button) + L582-623 (silent useEffect)
- `rolliworking/src/pages/settings/BackfillBandJobs.tsx` L99-144 + L146-193 (`fixBlankNames`)
- RW investigation report received during silo 9 (Section B + C of "Shop Floor / Backfill / Intake Diagnostic")
