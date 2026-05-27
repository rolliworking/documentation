---
silo: 7
date: 2026-05-14
status: visual verification only, no debugging value
last_validated: 2026-05-14
---

# EST-24027 — Jenny Davis (backfill button + shop floor render check)

## What it is

A job in RolliWorking with no components (`job_components` row count = 0) that Mike captured in a screenshot during silo 7's manager-access investigation.

## Identifying data

- **System:** RolliWorking
- **Estimate number:** 24027
- **Customer:** Jenny Davis
- **Watch:** Rolex 13mm Two Tone Oval Jubilee
- **Component state:** none ("No components on this job")

## Why it became a fixture

Used to verify two things in passing:
1. **Backfill "+" button still works for the user's role.** The "+ Backfill components" button visible in the screenshot is gated to owner/manager. Visibility confirmed manager-level access is intact.
2. **Shop floor map renders for non-owner accounts.** The shop floor visualization at the top of the screenshot was visible to Mike on the account he was testing, confirming that the original "managers can't access shop floor" framing was wrong.

## How to use it

Not a primary fixture. Useful only as visual evidence that:
- Backfill "+" button works under the tested role
- Shop floor renders under the tested role

If the manager-access question ever resurfaces, this is the proof that the route gating is not the problem.

## Value to future silos

Low. Documented mainly because it appeared in chat alongside the more substantive fixtures, and to avoid future confusion about why est-24027 was referenced.
