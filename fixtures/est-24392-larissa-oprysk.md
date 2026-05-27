---
silo: 8
date: 2026-05-15
status: still in production as test bed; silo 8 ship of pills validated on this estimate
last_validated: 2026-05-15
---

# EST-24392 (Larissa Oprysk) — Single-bracelet test bed

## Identifiers

- `estimate_number`: 24392
- `customer`: Larissa Oprysk
- `watch`: Rolex SS D Link Jubilee (bracelet repair)
- `inspection_id`: INS-01950 (status sent)

## DB state silo 8

```
jobs.status               waiting_approval (still — see Plan D backfill decision)
job_components            1 row:
  component_type          bracelet
  status                  in_progress
  station_id              band_tech   ← the 1 of 222 components with station_id
  department              band_repair
  assignee_name           Joseph (JV)
  assignee_initials       JV
```

~15 log rows in 24h on this single bracelet. **The bulk of this log activity is silo 7 drag-test churn, not real workflow.** Real bracelets won't have this volume.

## Why it's a fixture

This is the only component in production with `station_id` populated. Used as:
1. Test bed for silo 7 QC inspect drag (`band_tech` station)
2. Single-component case for silo 8 pills visual smoke test
3. One of 4 backfill candidates Plan D explicitly skipped
4. Reference data when investigating "how should pills look for a job with assignee_initials present"

## Inspection findings (from existing data)

> Bracelet — Aftermarket (not made by Rolex) · Rebuild Anyways? · not real rebuild anyways? · will be 1/2" to 5/8" shorter

Client approved 2:
- Bracelet: not real rebuild anyways? — $0.00
- Might be shorter than arrival length — 2 links @ $135 — $270.00

Client notes:
> Is it still stainless and 18k white gold? Is it worth rebuilding? Do you know where I can get a rolex jubilee band?

Q2 answer: 7

## Why bracelet is in_progress while job is waiting_approval

Workers started the work before client formally approved. Documented in handoff Section 4 as legitimate operational reality — system tolerates this contradiction by design. Plan D's trigger going forward would normally bump this to `in_progress` when the bracelet moved, but the backfill of pre-existing rows like this one was explicitly skipped.
