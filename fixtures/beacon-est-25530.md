---
silo: 9
date: 2026-05-25
status: ACTIVE — primary beacon for ongoing pipeline verification
last_validated: 2026-05-25
---

# Fixture: EST 25530 — Primary Pipeline Beacon

## What it is

The first beacon estimate created to trace data flow end-to-end across RS → RW → RC.

## Data shape

**Customer:**
- Name: test hui
- Email: mikebhui@icloud.com
- Phone: 8189254081

**Estimate:**
- estimate_number: 25530
- Line 1: entered_part_number = `[B]`, description = `[B] Tracking-Beacon-Description-Est5`
- Line 2: entered_part_number = `BR-Tracking-Beacon`, description = `Tracking-Beacon-Description-Est6`
- (Plus shipping line and empty padding rows)

**Drop-off:**
- Received: 2026-05-25 at 10:20 PM
- Bracelet pill selected

## Linked parts row (curated source of truth)

```sql
SELECT * FROM parts WHERE part_number = 'BR-Tracking-Beacon';
```

| Field | Value |
|---|---|
| part_number | BR-Tracking-Beacon |
| brand | Rolex |
| bracelet_model | Tracking-Beacon-Model |
| department | B |
| item_type | service |
| description | Bracelet Repair - Tracking Beacon |

## Linked join (estimate_line_items joined to parts)

SQL run during silo 9 confirmed:

| sort | entered_part_number | part_id | parts_brand | parts_bracelet_model |
|---|---|---|---|---|
| 1 | `[B]` | [B] marker UUID | null | null |
| 2 | `BR-Tracking-Beacon` | linked to BR-Tracking-Beacon UUID | Rolex | Tracking-Beacon-Model |

## Expected behavior at each pipeline stage

| Stage | Expected |
|---|---|
| /inventory/parts | Brand=Rolex, Model=Tracking-Beacon-Model on BR-Tracking-Beacon row |
| Estimate edit page | Lines 1 & 2 visible |
| /intake/receive-watch | Brand pill=Rolex highlighted, Model field shows "Tracking-Beacon-Model", Item Type=Band Only, Dept B |
| After Save | client_property row written with brand="Rolex", bracelet_description containing "Tracking-Beacon-Model" |
| RS push payload | Caret string contains "Rolex" at position 6, "Tracking-Beacon-Model" at position 17 |
| RW jobs | watch_brand="Rolex", bracelet_description="Tracking-Beacon-Model", dept_b=true |
| RW job_components | bracelet row with department="band_repair" (post Fix A+B) |
| RW shop floor lookup | Job appears with identity showing Rolex (Tracking-Beacon-Model) — pending UI fix |
| RC customer portal | Timeline shows 6-step band flow (Package Received → Inspection → Awaiting Approval → Awaiting Start → Polish/Clean → Shipping/Pick Up) |

## Current observed behavior (silo 9 close)

- /inventory/parts: ✓ correct
- Estimate edit: ✓ correct
- Estimate join SQL: ✓ correct
- **/intake/receive-watch: ❌ Brand pill blank, Model field blank** ← stuck here
- Subsequent stages: untested

## Use this fixture for

- Verifying any code change that touches receive-watch pre-fill
- Verifying RS push payload construction
- Verifying RW receiver field mapping (post Fix A+B verification)
- Verifying RC band-flow Phase 1 (sectioned DTO v2)
- Permanent regression test during rebuild — should pass at every milestone

## Related beacon

`BR-Tracking-Beacon2` was created later in silo 9 as a parallel beacon with `bracelet_model='Tracking-Beacon-Model2'`. Has not yet been assigned to a separate estimate. Available for future testing.

## Cleanup note

These fixtures are intentional test data. Do not delete from production unless explicitly cleaning up test data during rebuild migration.
