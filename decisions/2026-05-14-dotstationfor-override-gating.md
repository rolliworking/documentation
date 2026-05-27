---
silo: 6
date: 2026-05-14
status: shipped (per Lovable apply confirmation)
last_validated: 2026-05-14
---

# Decision — Gate dotStationFor pre-lane override on `status === "in_queue"`

## Context

`dotStationFor()` had unconditional override at top of function:
```ts
if (jobStatus === "waiting_approval") return "lock_pre_approval";
if (jobStatus === "in_queue") return "lock_pre_queue";
```

This pinned dots to pre-lane locks regardless of component state. After drag set component to `in_progress`, override still fired → dot stayed at lock.

Real-world example: est 24187, bracelet assigned to Joseph at band_tech in database, dot rendered at lock_pre_approval on screen.

## Options considered

- **(A) Remove override entirely** — Always route by component status + department.
- **(B) Gate override on component's own status === "in_queue"** — Only apply pre-lane lock if component itself hasn't started.
- **(C) Gate on assignee_name being null** — Only apply pre-lane lock if no tech assigned.

## Decision

**Option B.**

```ts
if (status === "in_queue") {
  if (jobStatus === "waiting_approval") return "lock_pre_approval";
  if (jobStatus === "in_queue") return "lock_pre_queue";
}
// else fall through to component-level rules
```

## Reasoning

- Preserves "not started yet" signal (dots that haven't moved yet sit in pre-approval/pre-queue zones — visible at a glance)
- Lets component state win once work actually begins
- Matches operational reality: workers DO start work on jobs that haven't been formally approved
- 3-line change in single function, no API changes

## What would change this

If shop policy changes to forbid work-before-approval, Option A becomes simpler and more enforcing.

## References

- `src/pages/ShopFloor.tsx` — dotStationFor function (lines 161-167 after fix)
- Real-world test: est 24187 (Pallotolo's, Rolex Two Tone D Link Jubilee)
