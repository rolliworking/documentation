---
silo: 6
date: 2026-05-13
status: decided, implemented in v1
last_validated: 2026-05-14
---

# Decision — Use manual pointer events for shop floor drag, not @dnd-kit or HTML5 drag

## Context

Shop floor needed drag-and-drop on SVG dots between stations.

## Options considered

- **(A) @dnd-kit** — Library. Not currently installed. Would be new dep.
- **(B) HTML5 native drag** — Used in old ComponentStatus.tsx. Known broken on SVG cross-browser (Chrome partial, Safari/Firefox no). Doesn't fire on touch devices.
- **(C) Manual pointer events** — No library. Use `getScreenCTM().inverse()` for SVG↔DOM coordinate conversion. Hit-test against STATIONS array.

## Investigation finding

Lovable reported: zero `@dnd-kit` imports in codebase. BandKanban has zero drag code (despite the name). Native HTML5 drag in ComponentStatus.tsx is the only drag pattern.

## Decision

**Option C — Manual pointer events.**

## Reasoning

- No new dependency
- SVG already owns coordinate math via viewBox; conversion is one helper
- Works natively in SVG (HTML5 drag doesn't)
- Simpler than fighting library coordinate-space mismatches
- Touch support possible later (pointer events fire on touch)

## What would change this

- If a second drag use-case appears with very different shape, @dnd-kit might earn its place
- If a contributor unfamiliar with custom pointer logic needs to extend the drag, library familiarity could matter

## References

- `src/pages/ShopFloor.tsx` — drag handlers (plan-mode approved, apply unverified)
- BandKanban inspection (silo 6 chat)
