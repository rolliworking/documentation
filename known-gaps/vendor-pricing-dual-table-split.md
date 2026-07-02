---
silo: 11
date_surfaced: 2026-05-31
status: open - read-side gap, data conflict blocks mechanical sync
last_validated: 2026-05-31
---

# Vendor Pricing Dual-Table Split

## What

Two parallel tables hold vendor-pricing data in RS production. They are nearly disjoint, actively conflict where they overlap, and the reorder engine reads from only one of them, silently ignoring the table that holds the majority of the data.

## The tables

| | `vendor_parts` | `part_vendor_prices` |
|---|---|---|
| Created | 2026-01-10 07:13, initial purchasing migration | 2026-01-10 19:07, 12 hours later, no comment / no FIXME / no backfill |
| Row count, prod 2026-05-31 | 100 | 1,539 |
| Price column | `vendor_cost numeric(12,4)` | `unit_cost numeric` |
| Written by | AutoPO, `PartsVendorsTab.tsx`, `AddVendorPartDialog`, `LowStockReportPage.tsx:181`, `VendorPartsImportPage` | `VendorPricingImportPage.tsx:160/419/426` only, CSV pricing import |
| Read by reorder engine | Yes - `generate_reorder_suggestions` reads `vp.vendor_id WHERE vp.is_preferred = true` | No - completely invisible to reorder suggestions |
| UNIQUE constraint | `(part_id, vendor_id)` | `(part_id, vendor_id)` |
| RLS | `is_authenticated()` | `is_authenticated()` |

## Overlap and conflict

Live production query, 2026-05-31:

| Metric | Value |
|---|---|
| `vendor_parts` `(part_id, vendor_id)` pairs | 100 |
| `part_vendor_prices` `(part_id, vendor_id)` pairs | 1,539 |
| Pairs in both tables | 7 |
| Of the 7 overlapping pairs, prices that disagree by more than $0.01 | **7 (100%)** |

The tables are essentially disjoint datasets that actively contradict each other where they intersect.

## Silent bug: reorder ignores 1,400+ parts' vendor pricing

`generate_reorder_suggestions` reads only `vendor_parts`:

```sql
LEFT JOIN public.vendor_parts vp
  ON vp.part_id = p.id AND vp.is_preferred = true
...
suggested_cost = COALESCE(vp.vendor_cost, p.last_cost, p.average_cost)
vendor_id      = vp.vendor_id
```

For the approximately 1,400 parts with pricing in `part_vendor_prices` but no row in `vendor_parts`, reorder suggestions emit `vendor_id = NULL` and fall back to `parts.last_cost`, then `parts.average_cost`. The UI renders these as "No vendor" rows in `ReorderAlertsPage.tsx:404`.

Operators have been CSV-importing vendor pricing that the reorder engine never sees. This has been silently broken since 2026-01-10 19:07, when `part_vendor_prices` was created without integration.

## Why this matters now

Silo 11 surfaced an operator request: show the vendor list with color-coded prices for each part needing reorder, green/red/black versus average. The investigation that confirmed feasibility also surfaced this dual-table split as the load-bearing blocker.

The reorder x vendor-comparison feature was scoped to union both tables at query time, dedupe by `(part_id, vendor_id)`, and let `vendor_parts.is_preferred` win conflicts. That feature build does not fix this gap; it only stops making it worse for one new surface. The gap remains for other surfaces, including PartsVendorsTab, AutoPO, and `generate_reorder_suggestions` itself.

## Why this was not a mechanical sync

A sync migration was considered during silo 11 and blocked by the 100% disagreement rate on the 7 overlapping pairs. There is no metadata to automate which source wins:

- `vendor_parts` has `last_purchased_at` and `is_preferred`
- `part_vendor_prices` has `last_updated`, but no provenance

Neither table records which import, operator, or workflow set the current value. A sync would require operator judgment per conflict pair, which was out of scope as a code task.

## Fix options

| Option | Description | Trade-off |
|---|---|---|
| A. Sync and deprecate one table | Pick a winner per conflict pair with operator judgment, backfill missing rows from the loser table, drop the loser, update all readers/writers. | Cleanest end state. Requires operator review of conflicts before automation. |
| B. Union at read time everywhere | All consumers query both tables. Conflicts surface in UI for cleanup. | Two writers, two sources of truth remain, but readers stop missing data. |
| C. Repoint imports to a single table | Change `VendorPricingImportPage` to write to `vendor_parts`, backfill rows from `part_vendor_prices`, then drop the second table. | Same conflict-resolution problem on the 7 disagreement pairs. |
| D. Status quo plus UI surfacing | Silo 11's reorder-feature build: union on the new reorder surface only, surface conflicts to operators, accept dual writers. | Does not fix the existing gap. AutoPO, PartsVendorsTab, and `generate_reorder_suggestions` still have silent blind spots. |

## Recommendation

Prefer option A when the team can get operator input on the 7 conflict pairs as a one-time data exercise, then deprecate `part_vendor_prices` and repoint the CSV importer.

Option D is the right immediate ship for the silo 11 reorder x vendor-comparison feature because it is narrower and can surface conflicts without pretending the data conflict is mechanically resolvable.

## Diagnostic SQL

```sql
-- Disjoint vs overlap stats
SELECT
  (SELECT COUNT(*) FROM public.vendor_parts) AS vp_rows,
  (SELECT COUNT(*) FROM public.part_vendor_prices) AS pvp_rows,
  (SELECT COUNT(*)
   FROM public.vendor_parts vp
   JOIN public.part_vendor_prices pvp
     ON pvp.part_id = vp.part_id AND pvp.vendor_id = vp.vendor_id) AS in_both;

-- Disagreement pairs
SELECT
  vp.part_id, vp.vendor_id,
  vp.vendor_cost AS vendor_parts_price,
  pvp.unit_cost AS part_vendor_prices_price,
  ABS(vp.vendor_cost - pvp.unit_cost) AS diff,
  vp.last_purchased_at,
  pvp.last_updated
FROM public.vendor_parts vp
JOIN public.part_vendor_prices pvp
  ON pvp.part_id = vp.part_id AND pvp.vendor_id = vp.vendor_id
WHERE ABS(vp.vendor_cost - pvp.unit_cost) > 0.01
ORDER BY diff DESC;

-- Parts invisible to reorder because they only have part_vendor_prices entries
SELECT COUNT(DISTINCT pvp.part_id) AS invisible_parts
FROM public.part_vendor_prices pvp
WHERE NOT EXISTS (
  SELECT 1 FROM public.vendor_parts vp
  WHERE vp.part_id = pvp.part_id
    AND vp.is_preferred = true
);
```

## Related

- `known-gaps/perf-report-snapshot-drift.md` - different drift problem on a different table, but same shape: writes go to one place, reads expect another.
- Silo 11 reorder x vendor-pricing investigation.
