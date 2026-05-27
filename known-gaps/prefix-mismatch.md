---
silo: 10
date_surfaced: 2026-05-27
status: surfaced — mostly dead code on production data
last_validated: 2026-05-27
---

# Service Code Prefix Mismatch / Dead Matchers

## What it is

Multiple code paths attempt to derive department or flow from service-code prefix matching. The matchers reference prefixes (`W-`, `B-`, `P-`, `PM-`, `BR-`, `POL-`) but production data uses different conventions, and the structured `service_subcategories` table that the SQL matchers query is essentially unpopulated.

## Where the matchers live

| Location | Pattern | What it reads |
|----------|---------|---------------|
| `get_job_profile` SQL (migration `20260502221531_...sql`) | `service_code ILIKE 'W-%'` / `'B-%'` / `'P-%'` / `'PM-%'` | `service_subcategories.service_code` via `estimate_line_items.service_subcategory_id` |
| `src/types/rolliworks.ts:38` `getDepartment` | `PM-` / `POL-` / `BR-` / `W-` | `code` argument string |
| `src/types/rolliworks.ts:55` `getFlowFromCodes` | Uses `getDepartment` | Service codes array |
| `src/pages/intake/ClientWatchEntryPage.tsx:777-785` `autoSelectDepts` | `PM-` / `POL-` / `BR-` | `entered_part_number` text |
| `src/lib/constants.ts:198-229` `deriveServiceCode` | Mix; reads `part.department` first, then `BR-` / `POL-` on part_number | `parts.department` + `parts.part_number` |
| `rolliworking/supabase/functions/rollisuite-intake/index.ts:487-492` `getDeptIntake` | `W-` / `BR-` / `POL-` / `PM-` | Service codes from caret payload |

## Why most of these never execute

### SQL prefix matchers — dead

`service_subcategories` is a 9-row table:

| code | name | service_type |
|------|------|--------------|
| B-1, B-2 | Jubilee/Oyster Bracelet Repair | bracelet_service |
| P-1 | No Crown Guard Case Repair | other_service |
| SH-1 | Watchmaker Room Time | other_service |
| SM-1 | Small Job | other_service |
| WM-1, WM-2, WM-3 | Modern/Vintage/Antique Movement Service | watch_service |
| WR-1 | Warranty Work | warranty_service |

**Notable: there is no `W-*` code in this table.** Watchmaker work uses `WM-*` which does not match `W-%`.

**More critical:** of 5,362 line items in the last 90 days, `service_subcategory_id` is populated on **0**. The FK exists but the write path never sets it. Every SQL `ss.service_code ILIKE` branch dereferences `NULL` and contributes zero signal to flow detection.

### TS helpers — dead or near-dead

- `getFlowFromCodes` in `src/types/rolliworks.ts:55` and the duplicate in `rolliworking/src/lib/service-code-routing.ts:29-40` — both **defined but never called** anywhere in their respective codebases.
- `getDepartment` is called by `autoSelectDepts` and `deriveServiceCode`. `autoSelectDepts` reads `entered_part_number` prefixes `PM-` / `POL-` / `BR-` only (not `W-` or `WM-`), so watchmaker work is never auto-flagged from line text.
- `getDeptIntake` on RW receiver — same prefix universe, same blind spots for `WM-`.

### Active signal: marker_letters only

The only flow signal `get_job_profile` actually uses in production is `marker_letters` — extracted from `parts` rows where `item_type='service_marker'`, matched case-insensitively against `entered_part_number` text. Markers in use: `[W]`, `[B]`, `[P]`, `[PM]`, `[WB]`, `[WP]`, `[WBP]`. Other markers exist in the parts table (`[PB]`, `[SJ]`) but are not mapped to any flow in the CASE statement.

## entered_part_number in real production data

The field is free-text. Top conventions in active use across 5,362 line items (last 90 days):

| Convention | Examples | Count |
|------------|----------|-------|
| Marker (parts.item_type='service_marker') | `[W]`, `[B]`, `[WB]`, `[P]`, `[WBP]` | ~247 |
| Memo placeholders | `MEMO.`, `Client`, `Memo-1`, `Memo-3`, `Tech-Matthew`, `----` | ~700+ |
| Shipping codes | `2day-OBW`, `2day-OBB`, `2day_IBB`, `2day-IBW` | ~900+ |
| Watchmaker service blurbs | `W-SERV-Level-1`, `RLX-SERV-Level-1`, `W-SERV-VINT-1`, `RLX-SERV-VINTAGE-LEVEL-1` | ~295 |
| Band repair templates | `BR-JUB-TT-PIN`, `BR-OY-PIN`, `BR-OVAL-JUB-ALL`, `BR-JUB-SS`, `BR-RIVET-OY` | ~516 |
| Polish templates | `POL-CAS-NS`, `POL-BR`, `CASE-POL-CHAMFER` | ~157 |
| Precious metals | `PM-Solid Gold Bracelet`, `Bracelet Repair - Solid Gold 14k/18k WG/YG PM` | ~34 |
| Watchmaker hour buckets | `WM-HR (Disassembly/Reassembly)`, `WM-HR W-M` | ~99 |
| Caliber / inventory part numbers | `3130-311.`, `3035-5009.`, `3135-data`, `1520-8069.` | ~400+ |

There is no canonical convention. The field drifted to ~7 overlapping schemes. Trying to "fix the matcher" against this field is fragile by design.

## What is actually working today

The marker-letters branch. Operators add `[X]` markers to estimate line 1, the parts row of `item_type='service_marker'` is matched, and `fallback_flow` in the SQL is set. 90-94% of post-rollout estimates have this; 10% don't (see `markerless-estimates.md`).

## Fix paths

This gap is mostly cosmetic — the dead code can be deleted without changing behavior. The real question is whether to:

1. Delete the dead matchers (cleanup, no behavior change)
2. Wire `service_subcategory_id` to actually be set on line items (data fix; partial — would need code changes to track it)
3. Add a `department` column to the `parts` table and stop trying to derive department from text patterns (durable; biggest backfill)
4. Live with markers as the only flow signal and address the 10% markerless gap separately

Decision deferred — coupled to markerless-estimates decision.

## Verification queries

```sql
-- All service_subcategories rows
SELECT * FROM service_subcategories ORDER BY service_code;

-- How many line items have service_subcategory_id populated?
SELECT 
  COUNT(*) as total,
  COUNT(service_subcategory_id) as linked
FROM estimate_line_items
WHERE created_at > NOW() - INTERVAL '90 days';

-- Distinct entered_part_number values, sorted by frequency
SELECT 
  entered_part_number, 
  p.item_type,
  COUNT(*) as occurrences
FROM estimate_line_items eli
LEFT JOIN parts p ON p.id = eli.part_id
WHERE eli.created_at > NOW() - INTERVAL '90 days'
GROUP BY entered_part_number, p.item_type
ORDER BY occurrences DESC
LIMIT 100;
```
