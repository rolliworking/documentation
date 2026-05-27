---
silo: 7
date: 2026-05-14
status: known architectural choice, not addressed
last_validated: 2026-05-14
---

# Performance Report — identity field snapshot drift

## The gap

Identity fields on `performance_verification_reports` are text snapshots, not joins:
- `watch_brand`, `watch_model`, `reference_number`, `serial_number`
- `client_first_name` (added silo 7), `client_last_name`
- `estimate_number`

All stored as plain text on the report row at save time. The preview (`PerformanceReportPreviewPage.tsx`) never re-fetches from the source tables.

`sales_order_id` IS a FK on the row, so the joins COULD be done. They aren't.

## Consequence

If source data is corrected after the report is generated — e.g. a watch reference number typo fixed in `watches`, a customer name updated in `customers`, an estimate number changed — the report still shows the old (potentially wrong) value indefinitely.

For a customer-facing branded artifact (this is — see "no delivery mechanism" gap), that's a quiet correctness hazard.

## Why it's this way (best guess from code)

Likely an intentional snapshot pattern: at the time the report is generated, capture what was on the watch/customer/estimate so the report is "frozen" at that moment. The downside is the drift.

## Possible mitigations

1. **Re-fetch from joins at preview time.** Use the FK `sales_order_id` to live-join customers/watches/estimates and render current values. Biggest change, but eliminates drift entirely.
2. **Warn on entry page if snapshot disagrees with source.** Less invasive. Open report, see "watch reference has been updated since this report was generated — refresh?" toast.
3. **Accept drift.** Document as known behavior, leave as-is.

## Status

Not addressed this silo. Identified during the Performance Report investigation but out of scope of the client_first_name addition.
