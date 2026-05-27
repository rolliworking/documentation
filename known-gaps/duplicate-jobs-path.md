---
silo: 9
date: 2026-05-25
status: open — Fix C in backlog
last_validated: 2026-05-25
---

# Gap: Duplicate Jobs Path in RW Receiver

## What happens

`rolliworking/supabase/functions/rollisuite-intake/index.ts` lines 431-434: when an existing job's `client_id` doesn't match incoming customer, receiver creates a brand-new job for the same estimate. Result: two `jobs` rows for the same `estimate_number`, components attached to the orphan with placeholder name.

## Trigger condition

- Push #1 (barcode scan, no `full_name`) → receiver writes `Client #<est>` placeholder customer (line 293-296) + Job A
- Push #2 (operator types real name in RS, re-saves) → receiver sees customer mismatch on `client_id` → creates new customer + Job B for same estimate
- Components attached to Job A (placeholder) appear as "blank client name" in BandKanban filters and queries

## Why it matters

- Two `jobs` rows for the same estimate confuses every query downstream
- `fixBlankNames` backfill in `BackfillBandJobs.tsx` L146-193 exists solely to re-point `job_components.job_id` to the correct job row
- Operator-facing surfaces show duplicates or blanks depending on which job they hit

## Resolution path

**Fix C (~15 lines):** Match by `estimate_number` + normalized email instead of strict `client_id`. When push arrives with real name for a job currently tied to a `Client #<est>` placeholder customer, UPDATE the existing job's `client_id`, `client_name`, `client_email` rather than creating a parallel job.

**Fix D (~3 lines):** Stop writing `Client #<est>` placeholder to `jobs.client_name`. Leave NULL until real name arrives. Eliminates the root cause that drives Fix C.

## Evidence

- `rolliworking/supabase/functions/rollisuite-intake/index.ts` L431-434 — duplicate-jobs creation path
- `rolliworking/supabase/functions/rollisuite-intake/index.ts` L293-296 — placeholder name write
- `rolliworking/src/pages/settings/BackfillBandJobs.tsx` L146-193 — `fixBlankNames` data repair
- RW investigation report during silo 9 (Section B4 "Why is backfill running nearly every time?")
