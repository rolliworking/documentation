---
silo: 10
type: permanent test fixture
location: rolliworks RS Supabase, parts table
last_validated: 2026-05-27
---

# BR-Tracking-Beacon — Permanent Diagnostic Anchor

## What it is

A sentinel-value row in the RolliSuite `parts` table, created during silo 9/10 debugging, intended to remain permanent.

## Schema values

```
part_number:     BR-Tracking-Beacon
brand:           Rolex
bracelet_model:  Tracking-Beacon-Model
department:      B
item_type:       service
```

All values are sentinel — unique strings unlikely to collide with real production data. The `Rolex` brand is real but combined with `Tracking-Beacon-Model` produces a uniquely identifiable combination.

## Purpose

When tracing the pipeline:
- Estimate → estimate_line_items → parts join → receive-watch form prefill
- Receive-watch form → client_property record
- RS → caret/JSON push → RW intake → jobs/watches/job_components
- RW UI rendering of brand/model/bracelet identity
- RC customer portal timeline

The beacon's distinctive string values (`Tracking-Beacon-Model`) act as a diagnostic dye. If "Tracking-Beacon-Model" appears in a downstream system, the data flowed from `parts.bracelet_model` correctly through that stage. If it's absent, that stage broke the flow.

## How to use

### Create a beacon test estimate

1. Create a new estimate via RS estimates page
2. Line 1: `entered_part_number = [B]` (or other marker letter; tests marker path)
3. Line 2: `entered_part_number = BR-Tracking-Beacon` (links to the curated parts row)
4. Customer: any test customer

### Trace through receive-watch

1. Navigate to `/intake/receive-watch?estimate=<number>` OR scan the estimate number in the form's Scan Estimate Barcode field
2. Expected results:
   - Item Type: Band Only (auto-selected from job profile)
   - Department: B (auto-selected)
   - Brand pill: Rolex (highlighted)
   - Model field: `Tracking-Beacon-Model`
   - "Previous Watches" pill: `BR-Tracking-Beacon`
3. Console (with silo 10 diagnostic logs in place):
   - `[PREFILL EFFECT ENTRY]` with `has_estimateData: true`
   - `[BEACON TRACE]` with `has_part_object: true`
   - `[MATCHER CHECK]` showing `full_filter_passes: true`
   - `BRACELET PART DETAIL` with `found: true, brand: "Rolex"`

If any of these are absent or wrong, the pipeline broke at the corresponding stage.

### Known coverage gaps

The beacon currently uses the `[B]` marker on line 1. This exercises the marker-letters path in `get_job_profile`. It does NOT exercise the markerless / service-code path (see `known-gaps/prefix-mismatch.md`).

A second beacon estimate (no marker, service codes `WM-1` + `B-2`) would exercise the markerless path. Not yet created — deferred pending decision on markerless-flow direction (see `decisions/2026-05-26-defer-markerless-flow-fix.md`).

## Existing test estimates using the beacon

| Estimate | Markers | Status | Notes |
|----------|---------|--------|-------|
| EST 25530 | `[B]` | Test only | Primary regression anchor for silo 10 fix |
| EST 25531 | `[B]` | Test only | Used during initial diagnostic |
| EST 25532 | `[B]` | Test only | Used during final verification |
| EST 25533 | None | Draft | Beacon test with no marker — included in markerless 15-sample (silo 10) |

## Don't delete

The beacon must persist in the `parts` table across:
- DB migrations
- Data cleanup operations
- Test data purges

If a future operation considers cleaning the parts table:
- Filter exclusion: `WHERE part_number != 'BR-Tracking-Beacon'`
- Or move to a protected `test_fixtures` schema (would require updating estimates that reference it)

## Naming convention for future beacons

If additional sentinel fixtures are added:
- `BR-Tracking-<Purpose>` pattern (e.g. `BR-Tracking-NoMarker`, `BR-Tracking-SplitFlow`)
- Document each in its own fixture file in this documentation repo
- Reference in `silos/silo-N.md` where they were created

## Related

- `decisions/2026-05-25-beacon-strategy.md` — full rationale for adopting beacons permanently
- `decisions/2026-05-25-bulldozer-methodology.md` — beacons are the primary input to bulldozer pipeline traces
