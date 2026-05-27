---
silo: 10
date_surfaced: 2026-05-26
status: surfaced — user-reported operational pain
last_validated: 2026-05-27
---

# Package-Receive Data Trickle Failure

## What the user said

> "the package receive/drop off module data is basically going into the trash. i could very well just tell me staff to stop wasting their time inputting data that just gets ignored."

That's the operational impact in the user's words. This document is the technical map of what's actually broken.

## The intended flow

1. Operator at drop-off / shipping receive station logs what physically arrived (tracking, carrier, received items)
2. Data lands in `package_arrival_scans` table — fields: `tracking_number`, `carrier`, `scanned_at`, `memo` (free-form components string), `matched_estimate_id`, `received_at`, `received_by`
3. When operator opens receive-watch for that estimate, the form pre-selects the Items Received pills based on what was logged
4. After save + label print, the data carries forward to RW

## What actually happens

### Stage 1: drop-off → package_arrival_scans
Works. `package_arrival_scans` is populated when operators do their job.

### Stage 2: package_arrival_scans → receive-watch UI
**Partially broken.** Confirmed via EST 24833 in silo 10:
- Banner displays: "Already received via fedex on May 18, 2026 · received:Watch head only · Items received have been pre-selected below — confirm or adjust"
- Bottom of form: Items Received pills (Watch Head, Complete Watch, Bracelet, Watch Case, Clasp, Bezel, Box, Papers) — **all blank**
- Banner promises pre-selection. UI doesn't deliver it.

The form reads `package_arrival_scans.memo` (at `ClientWatchEntryPage.tsx:303-319` direct query and `get_job_profile.received.components` at SQL RPC L251-264) and parses into `receivedComponents: string[]` via `setReceivedComponents`. PRESET_VALUES at L366-369 are the expected pill identifiers (`watch_head`, `complete_watch`, `bracelet`, etc.).

Failure mode: either the parse from `memo` to PRESET_VALUES doesn't match (string format drift) OR the pre-selection logic isn't writing to the state the pills read from. Not investigated in silo 10.

### Stage 3: receive-watch → client_property
Saves work. `client_property.received_bracelet`, `received_case`, plus a `[received:...]` tag in notes.

### Stage 4: client_property → RS→RW payload
Caret position 17 carries `receivedComponents` (pipe-joined). RW receives it.

### Stage 5: RW → tech screens
**Broken.** `jobs.received_components` is written on intake but **never read** by any RW UI. See `rw-data-blind-spots.md`.

The band tech does work on a job without seeing what physically arrived. The operator typed the component list at drop-off, the form (sometimes) pre-selects it, the operator confirms, it gets saved to client_property, pushed to RW, stored — and then no UI consults it.

## Specific evidence from EST 24833

| Field | Value |
|-------|-------|
| Drop-off banner | "Already received via fedex on May 18, 2026 · received:Watch head only" |
| Bottom Items Received | All pills blank |
| Item Type radio | Band Only (auto-selected) |
| Department | W (Watchmaker) + P (Polish Room) auto-selected |
| Warning | "Band Only job should not have Watchmaker department selected" |
| Customer | David Corcoran |
| Job profile banner | "BAND ONLY FLOW" |

What this estimate shows: the system found the drop-off receipt (banner text proves the data was read). The system computed a flow (BAND_ONLY). The system auto-selected departments. But the Items Received pills — the ground-truth input from the drop-off station — failed to populate.

The Watch Head Only banner is correct (no band physically arrived). The Band Only flow classification appears to contradict that — but on inspection it's a different question: the estimate's *scope of work* is band-only, while the *physical receipt* is watch-head-only. Two truths that don't agree, and the system has no rule for reconciliation.

## Two distinct bugs hidden here

### Bug A: pre-selection logic
The Items Received pills should be pre-selected from `package_arrival_scans.memo` parse, but aren't. Localized fix in receive-watch form logic.

### Bug B: missing reconciliation
The estimated work (band-only flow) and the actual receipt (watch head only) disagree. The system has no rule for what to do:
- Should Item Type flip to match physical receipt?
- Should the form warn operator and require manual reconciliation?
- Should the system create a partial-receipt state ("awaiting band arrival")?

These are product decisions, not just code bugs.

## Related gaps

- `rw-data-blind-spots.md` — even if pre-selection worked, RW doesn't render the data
- `markerless-estimates.md` — wrong flow classification compounds reconciliation problem

## Fix priority discussion

Both bugs deferred at silo 10 close pending:
1. Confirmation of Bug A reproducibility (one estimate is a data point, not a pattern)
2. Product decision on Bug B reconciliation rule

If the user proceeds with "the order ticket" framing, both bugs are high priority — the ticket isn't complete until physical receipt data is reflected and reconciliation rules exist.

## Investigation queries

```sql
-- Recent drop-off scans that should be feeding receive-watch
SELECT 
  pas.id,
  pas.tracking_number,
  pas.scanned_at,
  pas.memo,
  pas.matched_estimate_id,
  e.estimate_number
FROM package_arrival_scans pas
LEFT JOIN estimates e ON e.id = pas.matched_estimate_id
WHERE pas.scanned_at > NOW() - INTERVAL '14 days'
ORDER BY pas.scanned_at DESC;

-- For a specific estimate, what was logged at drop-off?
SELECT memo, scanned_at, receive_status, received_at 
FROM package_arrival_scans 
WHERE matched_estimate_id = (SELECT id FROM estimates WHERE estimate_number = '24833');

-- For the same estimate, what was saved on client_property?
SELECT received_bracelet, received_case, notes 
FROM client_property 
WHERE estimate_id = (SELECT id FROM estimates WHERE estimate_number = '24833');
```
