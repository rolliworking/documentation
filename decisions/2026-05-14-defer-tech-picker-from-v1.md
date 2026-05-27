---
silo: 6
date: 2026-05-14
status: v1 + v2 designed; apply unverified
last_validated: 2026-05-14
---

# Decision — Defer tech picker from drag v1, ship as v2

## Context

Original Chunk 3 scope: drag to all valid drop targets including `assign_*` stations.

`assign_*` stations require a tech selection. Drag to those without picker = ambiguous state.

## Options considered

- **(A) Build v1 with picker** — Single bigger ship.
- **(B) Defer picker; v1 safes only; v2 adds picker** — Two smaller ships.
- **(C) Reconsider whether safes-only is enough** — Maybe v1 is sufficient long-term.

## Decision

Initially B (defer). Mike escalated to "critical — drag without picker isn't useful for correction tool use case." Final: still B, but with explicit commitment to v2 picker as the next ship.

## v1 shipped

- 7 lock stations as drop targets
- Status + department writes to job_components
- Audit log to job_component_logs
- Desktop only, banner if touch

## v2 shipped (apply unverified)

- 5 work-node stations as drop targets (movement_service, refinish_case, refinish_band, band_tech, qc_inspect)
- TechPickerDialog component with role-filtered tech list
- Pre-select current assignee if exists
- Write status + department + assignee_name + assignee_initials
- Cancel paths (X, Escape, overlay click) → snap back, no DB write

## Reasoning

- Two ships beat one bigger ship for blast-radius control
- v1 ships fast (smaller scope), workers immediately test core drag UX
- Picker UX is its own design problem (which techs, pre-selection, cancel behavior)
- Real usage data from v1 informs picker design

## What would change this

If v2 picker had been pure scope-creep (no operational need), Option C would have been right. Mike's "drag is for corrections, RW is buggy" framing made picker essential.

## References

- `src/pages/ShopFloor.tsx` — v1 + v2 drag implementation
- `src/components/shop-floor/TechPickerDialog.tsx` — v2 picker component
