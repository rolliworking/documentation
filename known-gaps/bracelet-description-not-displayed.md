---
silo: 9
date: 2026-05-25
status: open — UI work pending after data pipeline is correct
last_validated: 2026-05-25
---

# Gap: `bracelet_description` Stored but Not Displayed on RW Operator UI

## Current state

`jobs.bracelet_description` is written by RW receiver (line 614 of `rollisuite-intake/index.ts`) when payload provides it. Column exists, data lands.

But it's only consumed by **customer-facing** ApproveInspection email flow (Fix 1 from earlier silos). Shop floor operators, work queue, BandKanban, JobCard — none display it.

## Why it matters

For band-only jobs, `jobs.watch_brand` and `jobs.watch_model` are often blank (no watch involved). Without `bracelet_description` displayed, operator sees a job row with no identifying detail — just `estimate_number` and `client_name`.

Example: Rolex Oval Jubilee bracelet repair on EST 24818 should show "Rolex (Oval Jubilee 20mm)" on shop floor. Today shows just "Rolex" or blank.

## Resolution path

Helper `formatJobIdentity(brand, model, braceletDescription)` in `src/lib/job-utils.ts`:

```ts
export function formatJobIdentity(
  brand: string | null | undefined,
  model: string | null | undefined,
  braceletDescription: string | null | undefined,
): string {
  const brandModel = [brand, model].filter(Boolean).join(' ').trim();
  const bracelet = braceletDescription?.trim();
  if (!brandModel && !bracelet) return '';
  if (!bracelet) return brandModel;
  if (!brandModel) return bracelet;
  return `${brandModel} (${bracelet})`;
}
```

Output examples:
- Watch only: "Rolex Datejust"
- Band only: "Rolex (Oval Jubilee 20mm)"
- Split: "Rolex Submariner (Oyster 20mm)"

Apply at:
- `src/pages/ShopFloor.tsx` LookupView L1423-1432 (identity header) and L1172-1199 (bulk-assign queue row)
- `src/components/workqueue/*` — wherever `watch_brand + watch_model` are rendered together
- `src/components/jobs/JobCard.tsx`

## Dependencies

This is pure UI. No data flow changes needed — `jobs.bracelet_description` already populated when RS push includes it. Independent of receiver fixes.

## Evidence

- `rolliworking/src/pages/ShopFloor.tsx` L1306-1311 — current `jobs` query selects `watch_brand, watch_model` but NOT `bracelet_description`
- `rolliworking/src/pages/ShopFloor.tsx` L1423-1432 — current identity header render
- RW investigation report (Section A4 + A5) — "Visible gaps: blank identity for band-only"
