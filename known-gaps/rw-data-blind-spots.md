---
silo: 10
date_surfaced: 2026-05-27
status: surfaced — open, requires multi-track work
last_validated: 2026-05-27
---

# RW Data Blind Spots — Fields Sent But Not Rendered

## What it is

Operator-entered data flows from RS receive-watch into RW's `jobs`, `watches`, and `job_components` tables. RW reads some of it on intake but never displays it on tech-facing screens. The shop floor, work queue, and band kanban all render only `watch_brand` + `watch_model`, even for band-only jobs where those fields are blank or stale and the operator-entered `bracelet_description` carries the actual information.

The user's framing: "the package-receive data is basically going into the trash." This gap is the proof point.

## Specific dead-end fields

| Field | Stored in | Read by RW UI? | Notes |
|-------|-----------|----------------|-------|
| `jobs.bracelet_description` | RW jobs table | **No** on shop floor / work queue / band kanban. Partial render in `ApproveInspection.tsx` and `InspectionReport.tsx` only. | Customer-facing approval shows it; tech doing the work doesn't. |
| `jobs.received_components` | RW jobs table (text[]) | **No** anywhere — verified via codebase search | Stored on every intake, never queried. |
| `jobs.received_bracelet`, `jobs.received_case` | RW jobs table (bool) | Only used in `rollisuite-intake/index.ts:571` to nudge flow detection (SPLIT vs STANDARD). No UI surface. | |
| `jobs.flow` | RW jobs table | **No** — write-only. Verified via codebase search. | RC timeline uses `get_job_profile.flow`, not `jobs.flow`. |
| Caret payload position 8 (`braceletModel`) | Sent by RS | RW skips at L257-258 with comment "no DB column" | Dead read; not stored anywhere. |
| Caret payload positions 10/11 (`receivedBracelet`, `receivedCase`) | Sent by RS | RW caret parser **skips** with comment "separate commit" | Defaults to `true` for both. Bulldozer Bug 11. |
| JSON `bracelet_model` field | Sent by RS | Parsed into local var at `rollisuite-intake/index.ts:209`, never written to any column | Dead. |

## Specific positive renders (for reference)

Shop Floor card and lookup display:
```ts
// ShopFloor.tsx:1308
.select('id, estimate_number, client_name, watch_brand, watch_model, status, dept_w, dept_b, dept_p, dept_pm')
```

Work Queue card display:
```ts
// WorkQueue.tsx:1828-1829 
{job.watch_brand} {job.watch_model}
```

Band Kanban card:
```ts
// BandKanban L1035
{job.estimate_number} · {job.watch_brand} {job.watch_model}
```

**A band-only job for a customer who sent only their bracelet** ends up with `watch_brand="Rolex"` (operator-set or scan-set) and `watch_model="Band"` (label wizard default when bandOnly). Band tech sees "Rolex Band" instead of "Steel Solid link Oyster" — the actual identifying info is in `bracelet_description` which the UI never queries.

## Why this matters operationally

The user said staff has been entering data at the drop-off / package-receive step that never reaches the receive-watch form's Items Received pills, and downstream never reaches the band tech's work surface. From the user's perspective: "I could very well just tell my staff to stop wasting their time inputting data that just gets ignored."

That's the cost of these blind spots — operator trust in the data layer erodes.

## Fix paths (estimated, not validated)

### Minimum viable
- Add `bracelet_description` to RW shop floor / work queue / band kanban `select()` queries
- Add conditional render: for band-only jobs (`flow === 'BAND_ONLY_FLOW'` or `service_type === 'bracelet_repair'`), display `bracelet_description` instead of `watch_brand watch_model`
- Estimated effort: 2-4 dev hours

### Cleaner
- Add `jobs.bracelet_model` (or `job_components.model`) as a structured column
- Map RS `formData.braceletModel` (from `parts.bracelet_model`) into it on intake
- Render in tech cards
- Estimated effort: 4-6 dev hours including schema migration

### Comprehensive
- Migrate the entire RS→RW payload from 18-position caret to JSON with named fields
- Define a complete data contract document (started but not committed in silo 10)
- Add `flow` as a first-class field RS computes and RW stores trusted
- Add `expected_components` / `received_components` reconciliation with `awaiting_arrival` status
- Estimated effort: 8-12 dev hours

## Related to other gaps

- `nine-setformdata-paths.md` — RS side has the data, fragmented across 9 write paths
- `markerless-estimates.md` — `flow` misclassification feeds into wrong component creation here
- `prefix-mismatch.md` — RW receiver re-derives flow using same broken matchers as RS

## Source: bulldozer report findings 7-12

This gap is a consolidation of bulldozer report items:
- Bug 7: RW receiver skips caret[8] braceletModel
- Bug 8: ShopFloor + WorkQueue ignore jobs.bracelet_description (and bracelet_model)
- Bug 11: Caret pos 10/11 (receivedBracelet/receivedCase) sent but ignored
- Bug 12: useSendToRolliworking stale alternate path (16 fields, missing braceletDescription)

Plus net-new findings from silo 10 research:
- `jobs.flow` is write-only
- `jobs.received_components` is write-only
- JSON `bracelet_model` parsed but never persisted
- `getFlowFromCodes` in both RS and RW is dead code
