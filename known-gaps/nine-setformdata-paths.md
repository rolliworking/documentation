---
silo: 10
date_surfaced: 2026-05-26
status: documented — refactor deferred
last_validated: 2026-05-27
---

# Nine `setFormData` Paths in ClientWatchEntryPage.tsx

## What it is

`ClientWatchEntryPage.tsx` has 9 separate code paths that write to `formData.brand`, `formData.model`, `formData.partNumber`, or `formData.bracelet_description`. They were added incrementally as different prefill / decode / reset / picker scenarios emerged. They have overlapping responsibilities and they fight each other.

Silo 10's debugging burned 90+ hours partly because reasoning about prefill required understanding which of the 9 paths was firing on any given render. The root fix (commit `f2e1c0ba`) didn't touch any of them — it fixed the upstream query so the main prefill path actually ran.

## The nine paths

Path numbering follows Cursor's inventory during silo 10 research.

### 1. Effect-177 — clear brand/model on band_only itemType
```
ClientWatchEntryPage.tsx:181-186
if (hasPrefilledRef.current) return;
setFormData(prev => ({ ...prev, brand: '', model: '' }));
```

### 2. modelReference decode — band-only brand
```
ClientWatchEntryPage.tsx:203-208
if (!manualReferenceOverrides.brand && (!formData.brand || formData.brand !== modelReference.brand)) {
  setFormData(prev => ({ ...prev, brand: modelReference.brand }));
}
```

### 3. modelReference decode — watch brand/model/caliber
```
ClientWatchEntryPage.tsx:226-233
if (shouldUpdateBrand || shouldUpdateModel || shouldUpdateCaliber) {
  setFormData(prev => ({
    ...prev,
    brand: shouldUpdateBrand ? modelReference.brand : prev.brand,
    model: shouldUpdateModel ? (modelReference.model || '') : prev.model,
    caliber: shouldUpdateCaliber ? ((modelReference as any).caliber || '') : prev.caliber,
  }));
}
```

### 4. Multi-item bracelet fallback
```
ClientWatchEntryPage.tsx:657-663
setFormData((prev) =>
  prev.bracelet_description?.trim()
    ? prev
    : { ...prev, bracelet_description: opt.label }
);
```

### 5. Main prefill effect (THE primary path, with `[BEACON TRACE]` instrumentation)
```
ClientWatchEntryPage.tsx:951-972
setFormData({
  estimateId: estimateData.id,
  estimateNumber: estimateData.estimate_number,
  brand: isBandOnly ? (braceletPart?.brand || '') : (watch?.brand || parsed.brand || ''),
  model: isBandOnly ? '' : (watch?.model || parsed.model || ''),
  ...
});
```

### 6. handleEstimateScan — manual lookup partial merge
```
ClientWatchEntryPage.tsx:1105-1114
setFormData(prev => ({
  ...prev,
  estimateId: data.id,
  estimateNumber: data.estimate_number,
  brand: watch?.brand || prev.brand,
  model: watch?.model || prev.model,
  ...
}));
```
**No curated matcher. No `bracelet_description` set.** This is why scan-only entry was broken until silo 10's URL-handoff fix.

### 7. Post-save multi-item reset
```
ClientWatchEntryPage.tsx:1434-1448
setFormData(prev => ({
  ...prev,
  brand: '', model: '', partNumber: '',
  bracelet_description: '',
}));
```

### 8. Post-save full reset
```
ClientWatchEntryPage.tsx:1455-1468
setFormData({ ...empty defaults });
```

### 9. Previous-watch picker
```
ClientWatchEntryPage.tsx:1550-1557
setFormData(prev => ({
  ...prev,
  brand: watch.brand,
  model: watch.model || '',
  partNumber: watch.reference_number || '',
}));
```

## Plus manual UI handlers

Not counted in the 9 but they also write to these fields:
- L2083 — RadioGroup → Band Only
- L2158-2168 — watch Part # input
- L2186 — watch Brand input
- L2198 — watch Model input
- L2234, L2256, L2279, L2303, L2324 — watch bracelet presets/input
- L2352, L2360, L2378 — band-only brand pills + Other input
- L2395, L2417, L2442, L2488 — band-only bracelet presets/input
- L2513-2517 — multi-band Select

## Why this matters

`POST-PREFILL formData` console log fires on ANY change to brand/model/partNumber/bracelet_description. Any of these 9 paths can fire it. During silo 10 debugging we initially thought "prefill is firing twice and clobbering itself" — that was wrong. Multiple separate effects were firing in sequence. Reasoning about which was responsible required a complete inventory, which wasn't built until very late in the silo.

If silo 10's `.eq()` → `.or()` fix turns out to mask deeper interaction bugs between these paths (e.g. an effect cleaning data that shouldn't be cleaned), the first diagnostic step is logging which path fired.

## Refactor proposal (deferred)

Consolidate paths 1, 2, 3, 5, 6 into a single curated prefill function:
- Input: `estimateData` (loaded from any source — URL param, scan, customer selector)
- Computes: curated brand/model/braceletModel/bracelet_description/partNumber
- Output: single `setFormData` call

Paths 4, 7, 8, 9 are non-prefill (multi-item, reset, picker) and stay separate.

Estimated effort: 4-8 dev hours. Risk: significant — touches the heart of the page that just got stable.

Defer until at least one more silo of stability proves the current architecture isn't masking other bugs.

## How to test future changes here

Use the diagnostic logs left in place:
- `[BEACON TRACE]` — proves the main prefill effect entered
- `[MATCHER CHECK]` — shows what each part row looked like to the matcher
- `BR LOOP` — confirms iteration over sortedLines
- `BRACELET PART DETAIL` — final matcher result
- `POST-PREFILL formData` — any path's effect on formData

If `POST-PREFILL` fires without `[BEACON TRACE]`, the change came from one of paths 1, 2, 3, 4, 6, 7, 8, 9 — not the main prefill. That fingerprint is the silo 10 diagnostic that worked.
