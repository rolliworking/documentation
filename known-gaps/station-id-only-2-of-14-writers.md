---
silo: 8
date: 2026-05-15
status: open
last_validated: 2026-05-15
---

# station_id is only populated by 2 of 14+ writers

## What

`job_components.station_id` and `job_component_logs.station_id` are set by ShopFloor.tsx's `commitDrop` (line 664/672) and `handleCommit` (line 1033/1045) only. Twelve other writers create or update components without touching station_id.

## Evidence

As of silo 8 audit:

- `job_components`: 221 of 222 rows have NULL station_id. The 1 populated is EST-24392's bracelet at `band_tech`, drag-tested during silo 7.
- `job_component_logs`: 130 of 136 rows have NULL station_id. The 6 populated are silo 7 drag-test rows from EST-24392.

By action type:
| action | total | with station_id | NULL |
|---|---:|---:|---:|
| status_change | 90 | 6 | 84 |
| backfill | 42 | 0 | 42 |
| assigned | 3 | 0 | 3 |
| created | 1 | 0 | 1 |

The 42 backfill rows are NULL by design (BackfillBandJobs creates components that didn't exist at intake; no station applies). The other ~88 are unpatched-writer NULLs.

## Writers identified silo 8

`job_components`:
- `ShopFloor.tsx:664` commitDrop — Yes
- `ShopFloor.tsx:1033` handleCommit — Yes
- `ShopFloor.tsx:1576` smart-create from drop — No
- `NewJob.tsx:694` auto-create bracelet — No
- `BandKanban.tsx:242, 503, 520, 556, 968` — No
- `ComponentStatus.tsx:557, 607, ~810` — No
- `BackfillBandJobs.tsx:110, 179` — No
- `StationScanner.tsx:176, 302` — No
- `component-creation.ts:258, 268, 279` — No
- `rollisuite-intake/index.ts:577, 585, 594` — No
- `rollisuite-change-order/index.ts:99, 141, 154` — No
- `rollisuite-so-fulfilled/index.ts:75` — No

`job_component_logs`:
- `ShopFloor.tsx:672` commitDrop — Yes
- `ShopFloor.tsx:1045` handleCommit — Yes
- `BandKanban.tsx:255, 507, 560, 982` — No
- `BulkAssign.tsx:667, 703` — No
- `ComponentStatus.tsx:978` — No
- `StationScanner.tsx:181, 309` — No
- `NewJob.tsx:706` — No
- `BackfillBandJobs.tsx:124` — No (correct by design)
- `component-creation.ts:306` — No
- `rollisuite-change-order/index.ts:104, 146, 166` — No
- `rollisuite-so-fulfilled/index.ts:80` — No

`rollisuite-intake/index.ts` (the RW edge function that handles RS intake) is the primary source of watch_head / case / bracelet rows.

## Implication for design

Any UI surface reading station_id must treat NULL as the dominant case, not the exception. Department fallback (`'band_repair'`, `'watchmaker_bench'`, etc.) is the primary display path. The Job History Lookup pills shipped silo 8 deliberately do not read station_id; they use department as the fallback below assignee_initials.

## Why deferred

Real project, not urgent. Department fallback works fine for the current single consumer (pills). station_id propagation across all writers would touch ~14 files including 3 edge functions, with corresponding schema considerations for change-order and intake payloads. Worth filing as a real project rather than chasing per-writer fixes.

## How to find this again

```sql
SELECT action, COUNT(*) AS total,
       COUNT(station_id) AS with_station_id,
       COUNT(*) - COUNT(station_id) AS null_station_id
FROM job_component_logs
GROUP BY action ORDER BY total DESC;
```

If the `with_station_id` ratio jumps after silo 8, someone has been patching writers (good). If it stays flat, the gap is still open.
