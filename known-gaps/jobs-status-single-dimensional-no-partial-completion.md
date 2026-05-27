---
silo: 8
date: 2026-05-15
status: open — architectural limitation, not a bug
last_validated: 2026-05-15
---

# jobs.status is single-dimensional — partial-completion not surfaced to RS

## What

`jobs.status` is a single enum field with values: `intake`, `inspection`, `waiting_approval`, `in_queue`, `uncased`, `in_progress`, `parts_approval`, `parts_on_order`, `in_testing`, `waiting_components`, `finished`.

A job has multiple `job_components` rows, each with its own status. Component statuses can diverge: bracelet `waiting_components` while watch_head `in_progress` is a legitimate state during bracelet-done-watch-pending workflows.

But `jobs.status` cannot represent "bracelet done, watch still in progress." It's one value. So RS, which reads `jobs.status` only for its customer-facing timeline, cannot show partial completion.

## Concrete impact (surfaced silo 8)

Mike planned to bulk-assign 30-40 bracelets to a safe `waiting_components` station — band work done, watch not. Effect on visibility:

- **RW Job History Lookup pills:** correctly show `Bracelet: waiting components — <assignee>` and watch_head's current state. Honest picture.
- **RW Shop Floor:** bracelet dots render at safe location. Honest picture.
- **RS customer-facing timeline:** no visible change. RS reads `jobs.status` only. `waiting_components` on a component doesn't bump `jobs.status`. The customer sees no progress signal even though band work is genuinely complete.

## Plan D does NOT fix this

Plan D's trigger only fires on `→ in_progress`. It bumps parent from pre-in_progress states up to in_progress when any component starts work. It does NOT handle "one component done, parent partially advanced" — there's no obvious target jobs.status value.

Possible targets, all problematic:
- `waiting_components` — but that's an existing value meaning "job waiting on something to come back," not "one of our components is done"
- A new value like `partial_complete` — would require RS UI changes to render it
- Surface per-component progress to RS directly — major RS architectural change, would break the `jobs.status`-as-contract model

## Why deferred

Not silo 8's problem to solve. Documented because:
1. The bulk-assign workflow will hit this and produce confusing customer-facing visibility (or rather, the absence of expected visibility).
2. Future RC design needs to know jobs.status alone is insufficient for component-level visibility.
3. RC will likely consume `job_components` directly, which the handoff doc has noted before but the architectural reason matters.

## Workaround

For the immediate bulk-assign: accept that RS won't reflect the band-done state. Customer-facing timeline will look unchanged. If a customer calls to ask "is my band done?", staff uses RW Job History Lookup pills (silo 8 ship) to give the actual answer.
