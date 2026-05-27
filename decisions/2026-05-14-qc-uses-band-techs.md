---
silo: 6
date: 2026-05-14
status: decided
last_validated: 2026-05-14
---

# Decision — qc_inspect tech picker uses band_repair pool

## Context

Tech picker v2 needed role filtering per station. Investigation found:
- `watchmakers` table → movement_service
- `department_technicians` with `department='band_repair'` (2 rows in prod)
- `department_technicians` with `department='refinishing'` (1 row)
- `department_technicians` with `department='precious_metals'` (1 row)
- `qc_inspect` station has `department='qc'` but **zero QC techs exist in database**

## Options considered

- **(A) Use band_repair pool** — Band techs do QC operationally.
- **(B) Merge band_repair + refinishing** — Larger pool, anyone can QC.
- **(C) Skip qc_inspect from picker whitelist** — Don't allow drag here yet.

## Decision

**Option A.** Confirmed by Mike: band techs do QC in this shop.

## Reasoning

Matches operational reality. Adding a separate QC tech role is unnecessary infrastructure for how the shop actually runs.

## What would change this

If shop hires dedicated QC techs, add `department='qc'` rows to department_technicians and switch the picker's filter.

## References

- `src/pages/ShopFloor.tsx` — STATION_TECH_DEPT mapping
- Silo 6 chat — Lovable investigation found no qc rows
