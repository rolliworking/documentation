---
silo: 7
date: 2026-05-14
status: known issue, not fixed
last_validated: 2026-05-14
---

# `commitDrop` hardcodes `assigned_to: null` in log insert

## The gap

`src/pages/ShopFloor.tsx` `commitDrop` (lines 645-689) inserts a `job_component_logs` row on every drag commit (around line 679). The insert hardcodes `assigned_to: null` regardless of which tech was picked in the TechPickerDialog.

Compare with the sibling handler `handleCommit` (lines 995-1060) which correctly writes `tech?.id` to the log row.

## Why it matters

Timelines or audit views reconstructed from drag-flow log rows cannot identify the tech who was assigned — only that a status_change happened. The tech identity only lives in:
- `job_components.assignee_name` / `assignee_initials` (current state, not history)
- The log row's `notes` field which says "Shop Floor drag with tech assignment" (presence/absence signal only)

This limits the fidelity of any future event-timeline view in Job History Lookup. Reassignment events ("Bracelet reassigned EG → JV") cannot be reliably reconstructed from logs alone.

## Constraint that complicates the fix

`job_component_logs.assigned_to` is `uuid` typed. Watchmaker and department_technician records have their own IDs in `watchmakers.id` and `department_technicians.id` — NOT auth user UUIDs. The existing pattern writes `null` and relies on `assignee_name` / `assignee_initials` + notes for audit trail.

Possible fixes:
1. Mirror what `handleCommit` does — write `tech.id` even though it's not an auth UUID. Already done in `handleCommit`, no schema rejection in practice.
2. Add a separate `assigned_dept_tech_id` or `assigned_watchmaker_id` column to logs (matches the pattern on `job_components`).
3. Accept the gap; populate from `notes` parsing.

## Status

Not addressed this silo. Prerequisite for full event-timeline in Job History Lookup. Low urgency standalone; would be unblocked alongside Job History Lookup work.
