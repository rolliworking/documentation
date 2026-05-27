---
silo: 7
date: 2026-05-14
status: investigated, design recommended, not built
last_validated: 2026-05-14
---

# Job History Lookup — no shop floor visibility

## The gap

`JobHistoryLookup` dialog (`src/components/jobs/JobHistoryLookup.tsx`) currently shows:
- Job intake / created / inspection / approval timeline
- Client replies (inspection approvals, parts approvals)
- Inspection findings
- Email events (sent_email_templates, last_update_email_sent)
- Parts requests
- Header chips: status, assigned watchmaker, due date, inspection state

It does NOT show:
- Current component station (where is the bracelet right now?)
- Current component assignee (who is working on it?)
- Component status history (when did it move? where?)
- Reassignment events

All this data exists in `job_components` (current state) and `job_component_logs` (history). The dialog just doesn't query or render it.

## Why it matters

When troubleshooting "where is this job?" or "what's been happening with this bracelet?", the Job History Lookup is the natural place to look — but workers/managers have to leave the dialog and go to ShopFloor or BulkAssign to see component-level data. Friction.

## Recommended smallest meaningful addition

**Per-component current-state block at the top of the dialog**, before the existing timeline. For each component on the job, show:
- Component type ("Bracelet" / "Watch head" / "Case")
- Current station (resolved via `dotStationFor()` or station_id lookup against STATIONS[])
- Current status (in_queue / in_progress / in_safe / etc.)
- Current assignee (assignee_name from `job_components`)

For multi-component jobs, render as a stacked list:
```
Bracelet: at QC inspect · JV
Watch head: at Movement Service · EG
```

One additional Supabase read (likely joins onto existing `job_components` table). No granularity decisions. No reassignment-fidelity issues (snapshot doesn't care about history). No `commitDrop` log fix needed first.

## Bigger version (deferred)

Add a filtered event timeline:
- Filter to `from_status != to_status` (skip lock-shuffles)
- Collapse repeats within N seconds
- Per-component label on each row
- Merge with existing timeline events, sort chronologically

Requires fixing `commitDrop` to write `assigned_to` first (see related gap). Multi-day project, not a quick add.

## Status

Investigation complete (silo 7). Design recommended. Not built. Mike planned to do this in a fresh chat next session to preserve context budget for design conversation.
