---
silo: 7
date: 2026-05-14
status: applied to test branch (B1+B2), verified working
last_validated: 2026-05-14
---

# QC inspect routing — B1 (schema column) over A (crash-only) or B2 (composite signal)

## Context

Dragging a dot to QC inspect threw a Postgres CHECK constraint violation: `new row for relation "job_components" violates check constraint "job_components_department_check"`.

Root cause: the `STATIONS[]` entry for qc_inspect declared `department: "qc"`. The `job_components.department` CHECK constraint allows only `watchmaker_bench`, `refinishing`, `band_repair`, `precious_metals`, `safe_storage`. `'qc'` was never legal.

Three options:

- **Option A (crash-only):** Change `department: "qc"` → `"band_repair"`. Drag stops crashing. But on next page reload, `dotStationFor()` doesn't know to route the dot to QC inspect (the existing logic routes all `band_repair` in_progress components to `band_tech`). Dot would appear to "escape" QC inspect after refresh.
- **Option B1 (schema column):** Add `station_id text` nullable column to `job_components`. Drag handler writes `station.id` to it. `dotStationFor()` reads it first, falls back to existing logic when NULL or unknown.
- **Option B2 (composite signal):** Use `department = 'band_repair' AND assignee in QC tech list` to detect QC inspect routing. Requires identifying QC techs (column on department_technicians, or list in code).

## Decision

**B1.** User chose explicitly via in-chat option picker.

## Reasoning

1. **Dot position is real persisted state.** It survives sessions, refreshes, devices. State this user-visible belongs in the database, not derived heuristically.
2. **Future stations benefit from the same column.** Chunk 4 (notes UI on dots) and Chunk 5 (detail panel) will want the same "where is this dot?" query. Adding station_id now costs once; adding it later costs once per future feature.
3. **`dotStationFor()` already has complex fallback chain.** Reading station_id first short-circuits cleanly for drag-touched components without modifying the legacy logic for everything else.
4. **Investigation confirmed zero ripple to other writers.** Scoping investigation (`job_components.department` full read/write map) showed ShopFloor.tsx is the SOLE writer of any value that could route a dot to QC inspect. No edge function, no scanner code, no other UI produces QC routing intent. So station_id stays localized.
5. **Option A creates a worse UX.** Workers drag a dot to QC inspect, refresh later, find it sitting at band_tech. They'd think the drag didn't save. Strictly worse than the crash (which at least makes the failure obvious).
6. **Option B2 is brittle.** Requires maintaining a QC tech list. If a QC tech ever does other band work, the heuristic breaks.

## What shipped

Two-part fix:

**B1 (initial):**
- Migration: `ALTER TABLE job_components ADD COLUMN station_id text` (nullable), same for `job_component_logs`.
- `STATIONS[]` qc_inspect: `department: "qc"` → `"band_repair"`.
- `handleCommit` (lines 995-1060): writes `station_id = station.id` to update + log insert.
- `dotStationFor()` refactored: checks `station_id && STATION_BY_ID.has(stationId)` first, returns early. Else falls through to existing logic.
- LookupDot select extended to include `station_id`.
- Dead `if (department === "qc")` branch removed.

**B2 (follow-up after testing revealed gap):**
- B1 patched only `handleCommit` (scan flow). Drag flow uses `commitDrop` (lines 645-689). `commitDrop` was NOT patched, so drag writes still didn't include station_id.
- Diagnostic SQL query revealed station_id was NULL on the dragged test row.
- Two-line patch: `commitDrop` now writes `station_id: hit.id` to the update object AND to the log insert.

## Verification

- Migration ran: confirmed via `information_schema.columns` query — both tables have `station_id text` nullable.
- Functional test: user dragged dot to QC inspect, picked tech, refreshed — user confirmed "its working" — dot persists at QC inspect.

## What would change the decision

- If multiple writers ever need to set station_id (e.g. Bulk Assign or scanner adds a QC routing path), the model still works — just needs more places to write it. No rearchitecture required.
- If station_id values ever become stale (e.g. a station is removed from STATIONS[] but rows still reference it), the `STATION_BY_ID.has(stationId)` check ensures graceful fallback to derived routing rather than a crash.

## Tech debt surfaced

The `commitDrop` / `handleCommit` asymmetry (see related gap) made this two-patch process necessary. Should unify into a shared helper before more shop floor work lands.
