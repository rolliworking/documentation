---
silo: 9
date: 2026-05-25
status: open — backlog items, not yet shipped
last_validated: 2026-05-25
---

# Gap: RS Push to RW Drops or Hardcodes Fields

## What gets dropped or hardcoded in the push

RS push from `rollisuite/src/hooks/useRolliworkingSync.ts:sendIntakeToRolliworking()` lines 99-141 sends 18-field caret-delimited string to RW. Issues:

| Field | Issue |
|---|---|
| `date` | Always `format(new Date(), 'yyyy-MM-dd')` (today) — user-entered `formData.dateReceived` is ignored |
| `notes` | Not sent. Saved to `client_property.notes` locally only |
| `serial_number` | Not sent in JSON branch (forced empty L202 of RW receiver) — falls back to `part_number` |
| `bracelet_model` | Sent at pos 9 but **dropped** by RW receiver with comment "no DB column" line 258 |
| `size`, `material`, `inspection_type` | Not collected on form, not sent |
| Customer info | Only `fullName`, `email`, `phone` sent — no customer id, address, company, last_name split |

## Why it matters

Operator-typed data in RS receive-watch silently fails to reach RW. Operator at shop floor sees blank/wrong fields. Forces operators to re-enter or backfill data.

## Resolution path (silo 9 backlog)

- **Send actual `formData.dateReceived`** in push instead of `today` — 1 line change
- **Send `formData.notes`** in push — add field 19 or pack into existing payload
- **Add `bracelet_model` column on RW jobs** OR fold into `bracelet_description` properly (currently only happens via `composedBracelet` parser, not as separate column)
- **Auto-create `inspections` row on intake** linked to job, pre-populated with `bracelet_description` and notes

## Evidence

- `rollisuite/src/hooks/useRolliworkingSync.ts` lines 99-141 — push construction
- `rolliworking/supabase/functions/rollisuite-intake/index.ts` line 258 — "no DB column" comment for bracelet_model
- RS investigation report received during silo 9 ("RW Intake Push Map") — Section 4 "Field presence checklist"
