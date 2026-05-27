---
silo: 9
date: 2026-05-25
status: shipped to test branch (RS commit 98ce8a36, RC commit cf0a56e)
last_validated: 2026-05-25
---

# Decision: RC Sectioned DTO v2 with Backward Compat

## Context

RC customer portal timeline currently shows the same 20 watch-specific steps for all jobs — including band-only jobs. Customers see grayed-out steps that will never happen (Uncased, Parts Approval, etc.). Operator-facing complaint: "we only have a map for process flow for watch works and polish jobs."

## Options considered

**Option A: Minimum band-only branching in `rc-timeline-status`**
- Add flow detection, branch to band-flow step list
- Pros: Small surgical change, ships fast
- Cons: Still maintains parallel step model duplicated from RW shop floor STATIONS

**Option B: Unified model — RC reads same data RW shop floor uses**
- Refactor to return component-track data sourced from `job_components` + `station_id`
- Pros: Single source of truth, naturally handles split flow
- Cons: Bigger change, touches RC renderer, affects existing UX

## Decision

**Phase 1: Option A now (shipping minimum-impact fix).**
**Phase 2: Option B during 2-3 month rebuild.**

## Phase 1 DTO design (shipped)

Sectioned timeline:
- `BAND_ONLY_FLOW` → 1 linear section (6 customer steps)
- `STANDARD_FLOW` → 1 linear section (existing 20-step watch logic unchanged)
- `SPLIT_FLOW` → shared linear (3 steps) + parallel `{watch, band}` tracks + convergence linear

```ts
interface PortalTimelineDTO {
  ok: true;
  version: 2;
  flow: Flow;
  estimate_number: string;
  updated_at: string;
  sections: TimelineSection[];
  
  // Backward-compat for v1 clients (no UI rewrite required)
  display: {
    current_step_label: string | null;
    next_step_label: string | null;
    secondary_current_label?: string | null;
  };
  
  // v1 fields preserved at top level
  current_step_label?: string | null;
  next_step_label?: string | null;
  completed_steps?: string[];
}
```

## Band flow step list

| `jobs.status` | Customer sees |
|---|---|
| intake | Package Received |
| inspection | Inspection |
| waiting_approval | Awaiting Approval |
| in_queue | Awaiting Start |
| in_progress | Polish/Clean |
| finished | Shipping/Pick Up |

For SPLIT_FLOW, band track starts at "Awaiting Start" (shares front-end Package Received → Inspection → Awaiting Approval with watch track via shared linear section).

## RC client change

`src/pages/portal/ClientConversation.tsx` ProgressView reads:
- `status.version === 2` → check for v2 fields
- `display.current_step_label`, `display.next_step_label` for primary badge
- `display.secondary_current_label` → render second badge below primary if non-null (SPLIT only)
- Falls back to v1 fields if version < 2 or missing

~24 lines added to ClientConversation.tsx.

## Convergence rules (server computes, not RC)

For SPLIT_FLOW convergence:
- Both tracks complete pre-assembly work → `final_assembly` active
- Job `status === 'finished'` OR SO/shipment exists → `shipping_pickup` completed

RC never sees station IDs — only customer labels.

## Why this design (vs alternatives)

| Approach | Rejected because |
|---|---|
| Flat `steps[]` with `track: 'watch' \| 'band'` tags | RC must re-group, infer parallel vs linear, handle convergence |
| Return raw `job_components` to RC | Duplicates ShopFloor mapping in browser; RC becomes "smart" client |
| Separate API per flow type | Three renderers, three test matrices |
| Only extend v1 `current_step_label` string | Can't show parallel state honestly ("In Progress" hides band work) |

## Phase 2 plan (deferred to rebuild)

When timeline UI gets full redesign:
- `ParallelTimeline` component renders two columns, each reuses `LinearTimeline`
- RC reads `sections` array and renders generically; no flow-specific logic in React
- Server stays sole owner of flow-aware mapping (RW state → customer steps)

## Conditions that would change the decision

- If RC needs to render station-level detail (would push toward Option B sooner)
- If split-flow becomes >30% of job mix (would justify Phase 2 acceleration)
- If reuse-first surfaces a STATIONS-compatible renderer that already lives in RC (would reconsider Option B even for Phase 1)

## Evidence

- RS commit `98ce8a36` on test branch
- RC commit `cf0a56e` on test branch
- Cursor's design proposal (silo 9 transcript) — "SPLIT_FLOW DTO Recommendation"
- Both sanity checks passed (pre-intake flow detection defaults safely; `secondary_current_label` is null/undefined, never empty string)
