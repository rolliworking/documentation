---
silo: 6
last_validated: 2026-05-14
---

# Fixture — Estimate 24800 (component-less job)

## Identity

- Customer: Michael (Test)
- Watch: Rolex Two Tone D Link Jubilee
- Estimate number: 24800

## Why it's a fixture

Job exists with valid customer + watch but zero rows in `job_components`. Represents the "component-less job" failure mode (intake didn't create components, or this job predates component tracking).

## What it validates

- Backfill "+" feature in Component Lookup
- Pre-check logic driven by jobs.dept_* flags
- "Strict mode" (only show + when components.length === 0)

## Diagnostic value

When testing changes to:
- Component Lookup render logic
- Backfill creation flow
- dotStationFor with no components
- Job-has-no-components edge cases

...search for 24800 in Component Lookup. Expected: "No components on this job" + "+" button (if logged in as owner/manager).

## State after silo 6

Lovable claimed end-to-end test on 24800 in preview: clicked +, pre-check showed Bracelet only (dept_b=true), confirmed, bracelet component created with status=in_queue (per dotStationFor lock_pre_approval routing). Not independently verified in production.
