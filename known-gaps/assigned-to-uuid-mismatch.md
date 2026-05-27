---
silo: 6
discovered: 2026-05-14
status: accepted (existing pattern documented)
last_validated: 2026-05-14
---

# Gap — job_component_logs.assigned_to expects UUIDs that don't exist

## The problem

`job_component_logs.assigned_to` column is typed `uuid`. The actual assignment data lives in `watchmakers.id` and `department_technicians.id`. These tables are NOT in `auth.users` — they have their own IDs which are uuids in format but don't correspond to anything in auth.

Cannot write `assigned_to = <watchmaker.id>` without a type mismatch / FK violation.

## Existing pattern

`BulkAssignView` (and tech picker v2 followed this) writes:
- `assigned_to: null`
- `assignee_name: <tech name>` (string)
- `assignee_initials: <tech initials>` (string)
- `notes: "..."` (string, contains context)

Audit trail lives in name + initials + notes, not in the typed FK.

## Why this is gap, not bug

The schema implies a proper FK relationship that doesn't exist in practice. Anyone reading the schema cold would expect `assigned_to` to be populated. They'll write it that way, and it'll silently fail or break.

## Mitigation

Document the pattern. New code follows existing convention. Don't try to populate assigned_to.

## Permanent fix (not planned)

Either:
- Add FK to a real auth.users row (not all techs have accounts)
- Drop the assigned_to column and rely on string fields
- Add separate `assignee_table` + `assignee_id` columns for polymorphic ref

None scheduled.

## References

- `src/pages/ComponentStatus.tsx:978` — original pattern (writes assigned_to: null)
- Tech picker v2 commitDrop (silo 6) — follows same pattern
