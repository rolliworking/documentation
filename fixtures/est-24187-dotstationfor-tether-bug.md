---
silo: 6
last_validated: 2026-05-14
---

# Fixture — Estimate 24187 (dotStationFor override tether)

## Identity

- Customer: Pallotolo's Watches and Jewelry
- Watch: Rolex Two Tone D Link Jubilee
- Estimate number: 24187
- Job status: `waiting_approval` (per Lovable DB read)
- Bracelet component: status=`in_progress`, department=`band_repair`, assignee=Joseph (JV)

## The bug it exposed

After drag assigned the bracelet to Joseph at band_tech, the dot remained pinned at `lock_pre_approval` on the map. Component data showed in_progress; render showed pre-lane lock.

Diagnosis: `dotStationFor()` had unconditional override returning `lock_pre_approval` whenever `jobs.status = 'waiting_approval'`, regardless of component state.

## What it validates

- dotStationFor override fix (decision: gate on `status === "in_queue"`)
- Drag-to-work-node workflow (component state should follow drag, not job state)
- "Work can start before formal approval" operational reality

## Mike's note

Mike said "est 24187 has been approved" but `jobs.status = 'waiting_approval'`. Suggests either (a) approval workflow has a gap that didn't transition the job, or (b) "approved" meant inspection-approved not job-approved. Open question for separate investigation.

## State after silo 6

dotStationFor fix shipped (Lovable apply confirmation; Mike to manually verify by refreshing 24187 in production).
