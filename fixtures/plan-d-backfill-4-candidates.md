---
silo: 8
date: 2026-05-15
status: backfill skipped per Mike decision
last_validated: 2026-05-15
---

# Plan D backfill candidates — 4 jobs in waiting_approval with bracelet:in_progress

## The 4 jobs

- EST-24800
- EST-24185
- EST-24392 (Larissa Oprysk — see separate fixture)
- EST-24187

## Audit query

```sql
SELECT j.status AS current_job_status, COUNT(DISTINCT j.id) AS job_count
FROM public.jobs j
WHERE j.status IN ('intake','inspection','waiting_approval','in_queue','uncased')
  AND EXISTS (
    SELECT 1 FROM public.job_components c
    WHERE c.job_id = j.id AND c.status = 'in_progress'
  )
GROUP BY j.status
ORDER BY job_count DESC;
-- Result silo 8: waiting_approval = 4 (only category).
-- intake, inspection, in_queue, uncased all = 0.
```

## Common shape

All 4:
- `jobs.status = 'waiting_approval'`
- Have exactly one component (bracelet) in `status='in_progress'`
- `work_started_at` is NULL on the parent jobs row
- Are band-only or have started band work pre-approval

## Why backfill was skipped

Plan D's trigger going forward would normally bump these to `in_progress`. The one-time backfill was offered as part of the plan. Mike rejected backfill.

Reason: bumping these from `waiting_approval` to `in_progress` would push to RS in_progress, telling the customer (via RS timeline) that work has started, before they've formally approved. That's not the message Mike wants to send to the customer.

The trigger going forward IS allowed to bump from waiting_approval (Mike OK'd that for future activity). The backfill specifically is skipped because surfacing the existing pre-approval work would be misleading.

## What happens to these jobs

They stay in `waiting_approval` with `bracelet:in_progress` until:
1. Customer approves → normal flow advances to `in_queue` / `in_progress`, OR
2. Customer declines → job cancelled, OR
3. Some other manual transition

The bracelet component stays at `band_tech` (or wherever it moves to) regardless of parent jobs.status. RW pills (silo 8 ship) will show the bracelet's true state. RS timeline will continue showing pre-In Progress until the trigger fires on a future component transition.

## EST-24187 specifically

Mike flagged silo 7 that EST-24187 was "approved per client but jobs.status = waiting_approval." Status mismatch still open. Possible:
- Approval didn't transition the job (bug)
- Approval was logged but the transition action wasn't taken
- Approval was verbal / out-of-band and never recorded in `inspection_approvals`

Worth investigating separately if the customer asks about progress. Not addressed silo 8.
