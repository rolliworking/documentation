---
silo: 9
date: 2026-05-25
status: ACTIVE — secondary regression case
last_validated: 2026-05-25
---

# Fixture: EST 25523 — Rolex Oyster (BR-OY-PIN) Regression Case

## What it is

Second band-only test case discovered during silo 9. Used to verify the receive-watch fix wasn't specific to EST 24818's data shape.

## Data shape

**Customer:** mike (unknown surname, test customer)

**Estimate:**
- estimate_number: 25523
- Line 1: `[B] 20MM Rolex 93150 Oyster -`
- Line 2: part_number `BR-OY-PIN`, description `Bracelet Repair - SS Bracelet Repair. Remove stretch...`

## Linked parts row (curated)

```sql
SELECT * FROM parts WHERE part_number = 'BR-OY-PIN';
```

| Field | Value |
|---|---|
| part_number | BR-OY-PIN |
| brand | Rolex |
| bracelet_model | Steel Solid link Oyster |
| department | B |
| item_type | service |

## Notable other BR-OY-* parts (NOT all curated)

```sql
SELECT part_number, brand, bracelet_model
FROM parts WHERE part_number ILIKE 'BR-OY%';
```

| part_number | brand | bracelet_model |
|---|---|---|
| BR-OY-FOLDED-SS-TT | Rolex | SS Folded Oyster |
| BR-OY-PIN | Rolex | Steel Solid link Oyster |
| BR-OY-PIN-SS | null | null ← NOT CURATED |
| BR-OY-PIN-TT | null | null ← NOT CURATED |
| BR-OY-TT-PIN | Rolex | Two Tone Solid link Oyster |

## Expected behavior on receive-watch

- Brand pill: **Rolex** highlighted
- Model field: "Steel Solid link Oyster"
- Item Type: **Band Only**
- Department: **B**

## Status

Tested during silo 9 — showed blank Brand pill at first, leading to beacon investigation that surfaced the deeper bug pattern.

## Use this fixture for

- Verifying fix works for "real production" Rolex band jobs (vs synthetic beacon)
- Testing whether different `bracelet_model` strings render correctly in the Model field
- Regression check after any band-only pre-fill changes
