---
silo: 7
date: 2026-05-14
status: active fixture, used as test bed multiple times this silo
last_validated: 2026-05-14
---

# EST-24392 — Larissa Oprysk bracelet repair (QC inspect test bed)

## What it is

A bracelet-repair job in RolliWorking used as the test bed for the QC inspect crash fix (B1+B2), and later as the example fixture for Job History Lookup investigation.

## Identifying data

- **System:** RolliWorking
- **Estimate number:** 24392
- **Customer:** Larissa Oprysk
- **Watch:** Rolex SS D Link Jubilee
- **Type:** bracelet repair (band-only flow)
- **Job status at silo close:** waiting_approval (per Job History Lookup screenshot)
- **Due date:** Jun 3, 2026
- **Inspection:** INS-01950, completed 2026-05-13, status `sent`
- **Client approval:** approved 2026-05-14 (with notes asking about stainless/18k question)

## Why it became a fixture

This was the job displayed on Mike's screen when he tried to drag a dot to QC inspect and got the constraint violation error: `new row for relation "job_components" violates check constraint "job_components_department_check"`. The screenshot of that error became the trigger for the entire QC inspect investigation.

After B1+B2 shipped, the same job was used to verify the fix worked.

Later, this job was used as the example fixture in the Job History Lookup investigation since:
- The dialog screenshot Mike provided was specifically of this job
- The shop floor test churn from silo 7 left ~15 `job_component_logs` rows for the single bracelet component
- This provided concrete sample data to design the lookup UI against

## DB state at silo close

`job_components`:
- One row: bracelet, status `waiting_components`, department `band_repair`, station_id `lock_safe_awaiting`, assignee JV/Joseph, status_changed_at 2026-05-14 01:22

`job_component_logs`: 15+ rows ordered by `created_at`. Latest few:
```
2026-05-14 18:24:21  status_change  in_queue→in_progress  band_repair  "Shop Floor drag with tech assignment"
2026-05-14 18:24:26  status_change  in_progress→in_safe   (no dept)    "Shop Floor drag"
2026-05-14 18:24:34  status_change  in_safe→in_progress   band_repair  "Shop Floor drag with tech assignment"
... (rapid back-and-forth — 7 events in 16 minutes)
2026-05-15 03:24:10  status_change  in_progress→in_safe   station_id=lock_post_band       "Shop Floor drag"
2026-05-15 03:24:22  status_change  in_safe→in_safe       station_id=lock_pre_refinish_band "Shop Floor drag"
2026-05-15 03:24:25  status_change  in_safe→in_progress   refinishing  station_id=refinish_band "Shop Floor drag with tech assignment"
```

Two `changed_by` users active during testing: Michael Hui and Ivy P.

## Caveat for future readers

The 15+ log rows are mostly **testing churn from silo 7**, not real workflow. A real bracelet wouldn't have this many in 24h. When using this job as a fixture for Job History Lookup UI design, discount the testing rows (anything between roughly 2026-05-14 18:24 and 2026-05-15 03:24).

## How to use it

**For QC inspect drag verification:**
- Open `/component-lookup`, search for 24392.
- Drag the bracelet dot to QC inspect, pick a tech.
- Refresh page. Dot should stay at QC inspect.
- Without the fix, dot routes back to band_tech.

**For Job History Lookup UI design:**
- Sample data is rich enough to test merged-timeline rendering.
- Single-component (bracelet only, no watch_head) — useful for the simpler case.
- For multi-component testing, find a different job (silo 7 used `0226ecc7-...` as a multi-component example).

## DB query to inspect

```sql
-- Component state
SELECT * FROM public.job_components
WHERE job_id = (SELECT id FROM public.jobs WHERE estimate_number = '24392');

-- Activity history
SELECT created_at, action, from_status, to_status, department, station_id, notes, changed_by
FROM public.job_component_logs
WHERE job_id = (SELECT id FROM public.jobs WHERE estimate_number = '24392')
ORDER BY created_at DESC;
```
