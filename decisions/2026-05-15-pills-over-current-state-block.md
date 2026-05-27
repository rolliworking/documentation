---
silo: 8
date: 2026-05-15
status: shipped to RW production
last_validated: 2026-05-15
---

# Decision — Component status pills instead of current-state block in Job History Lookup

## Question

How should the Job History Lookup dialog surface "where is this job and who's working on it" without a separate Shop Floor lookup trip?

## Options considered

1. **Current-state block** (Lovable's silo 7 recommended design) — Section above timeline titled "Shop floor," one stacked card per component with status / station / assignee, NULL station_id handled via department fallback. New visual element, ~30 lines of new JSX inside the dialog.

2. **Per-component pills below existing header pills** (Mike's redirect) — Reuse the existing `<Badge>` component already rendering the 3 header pills (status / Due / Inspected). Add one pill per component with status + assignee_initials (or department fallback).

3. **Just update the existing 3 pills to be smarter** — Make the leftmost `jobs.status` pill more informative. Smallest possible change.

4. **Deep-link to Shop Floor** — Add a button to the dialog that navigates to `/shop-floor?estimate=<n>` with the lookup pre-loaded. No data added to the dialog.

## Chose: Option 2

Pills design. One pill per component below the existing 3.

## Why

- **Tighter than option 1.** No new visual element category — just more of an existing pattern. Fits the dialog's narrow `max-w-md` column without scroll pressure.
- **More informative than option 3.** A single `jobs.status` pill can't represent multi-component states (bracelet done, watch in progress).
- **Doesn't sidestep the surface like option 4 does.** Mike wants the answer in the dialog, not via navigation. Deep-link still valuable as a follow-up but doesn't replace this.
- **Honest about station_id NULL.** Option 1 was designed pre-silo-8-audit and assumed station_id would populate over time. Pills as shipped don't read station_id at all — only department + assignee. Matches the operational reality (221/222 NULL) instead of designing around an aspiration.

## Trade-offs accepted

- Multiple pills on small viewports may wrap awkwardly for 3-4-component jobs. Acceptable — most jobs have 1-2 components in practice.
- Pills don't show "which station." Future improvement when station_id is reliably populated. Department-level is honest now.
- No timeline of past component movements. That's the bigger "filtered event timeline" project; deferred.

## What would change this decision

- If station_id gets populated for >50% of components (would require patching the 12 other writers), pills could be enhanced to show station label when available.
- If multi-component jobs become common enough that pill-wrap UX is bad, switch to the stacked-card option 1 design.
- If RC needs the same data, both pills and the underlying query are reusable.

## Implementation

`src/components/jobs/JobHistoryLookup.tsx` only:
- New `useState<ComponentRow[]>` for components
- New SELECT against `job_components` in `selectJob`, filtered to `status != 'cancelled'`
- Sort + map in header JSX, reusing `Badge variant="secondary" className="text-xs"`
- Fallback precedence: `assignee_initials` → titlecased `department` → status alone
