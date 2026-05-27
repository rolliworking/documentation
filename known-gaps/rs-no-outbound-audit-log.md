---
silo: 6
discovered: 2026-05-14
status: deferred (Plan B)
last_validated: 2026-05-14
---

# Gap — RS does not persist outbound RW intake payloads

## The problem

`sendIntakeToRolliworking` in `src/hooks/useRolliworkingSync.ts` fires the payload to RW's `rollisuite-intake` endpoint and discards the result. No `rw_sync_log`, no audit table, no row in any persistent store.

Verified empty for est 24185 — could not reconstruct what was sent.

## Why it matters

- Cannot debug historical intakes ("what did we send for E#####?")
- Cannot retry failed pushes (no record of what to retry)
- Cannot prove what RW received vs what RS sent
- Cross-system blame in disputes is impossible

## Plan B (deferred)

New table `rw_intake_log`:
- id, estimate_number, payload_text, response_status, response_body, sent_at, sent_by

`sendIntakeToRolliworking` writes a row before fetch, updates with response on completion. Fire-and-forget UX preserved.

## Why deferred

Plan A was higher priority (fixed actively-wrong data). Plan B is the safety net for everything else.

## References

- `src/hooks/useRolliworkingSync.ts:69-97` — current fire-and-forget implementation
- Investigation in silo 6 chat (24185 data trail attempt)
