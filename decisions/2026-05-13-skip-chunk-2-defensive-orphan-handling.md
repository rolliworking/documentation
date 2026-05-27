---
silo: 6
date: 2026-05-13
status: decided
last_validated: 2026-05-14
---

# Decision — Skip Chunk 2 (defensive orphan/cancelled handling) entirely

## Context

Chunk 2 of shop floor redesign was scoped as defensive hardening:
- Render orphan job_components rows
- Handle null station returns from dotStationFor
- Handle cancelled components specially
- Verify color logic

## Investigation finding

Read-only queries against production database:

| Scenario | Count |
|---|---|
| Orphan job_components (FK drift) | 0 |
| Jobs with no estimate AND no serial | 0 |
| Cancelled components | 0 |
| dotStationFor null returns | 0 (function returns string in all branches) |

Color logic confirmed: `DOT_COLORS` encodes component type (blue/purple/green).

## Decision

Skip Chunk 2 entirely. Move to Chunk 3 (drag).

## Reasoning

Building for defensive cases that don't exist is theoretical work. "Iterate after build" principle applied: don't preemptively harden against zero data.

## What would change this

If real orphan or cancelled-component data appears in future, revisit. Watch for it via periodic count queries.

## References

- Silo 6 chat — investigation report from RW Lovable
- Aging-color overlay rejected separately (Daily Hit List handles)
