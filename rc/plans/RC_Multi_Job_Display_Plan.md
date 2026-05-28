# Plan — Multi-Job Progress Display

**Captured:** Sunday May 17, 2026
**Target build:** Memorial Day weekend
**Estimated effort:** ~3-4 hours cross-system

## The problem

Customers often have multiple active jobs (different estimate numbers), but the current Progress display shows ONLY ONE — other jobs are invisible from this view.

Example: Drew Hays has Estimate #24932 and #24938 both active. Staff opening his conversation only sees #24932. They have no idea #24938 exists.

## The vision

Show the customer's COUNT of active jobs prominently, with an accordion to switch between them. Same pattern on both desktop and mobile.

**Header line:**
```
Drew Hays · 2 active jobs
drew.hays1@gmail.com · 5126362936
```

**Single accordion pattern for both platforms:**

```
Drew Hays · 2 active jobs

▼ Estimate #24938 (newest)
   ✓ Estimate Sent → Customer Approval

▷ Estimate #24932
   (collapsed — click to expand)
```

- Default expanded: most recent estimate # (newest)
- Click/tap a collapsed header → expands it, collapses the previously open one
- Only one expanded at a time
- Single-job customers: always expanded, no collapse interaction (chevron hidden)

## Design decisions

| Decision | Value |
|---|---|
| Default selection | Most recent estimate # (newest) shown expanded |
| Indicator when multiple jobs | Count in header: "X active jobs" |
| Switcher | Accordion (same pattern on desktop + mobile) |
| Single-job customers | Always expanded, no switcher visible |
| Identification | Estimate # only (compact) |
| Sort order | Most recent first |
| Visual indicator | ▼ expanded, ▷ collapsed |

## Architecture changes

### RS endpoint update (~1 hr)

Current `rc-timeline-status`:
- Returns: ONE estimate's status

Needed: returns ALL active estimates for a customer

**New response shape:**
```json
{
  "ok": true,
  "estimates": [
    {
      "estimate_number": "24938",
      "current_step": "Estimate Sent",
      "current_step_label": "Estimate Sent",
      "completed_steps": ["Estimate Created", "Estimate Sent"],
      "next_step": "Customer Approval",
      "next_step_label": "Customer Approval",
      "updated_at": "2026-05-17T..."
    },
    {
      "estimate_number": "24932",
      "current_step": "Intake Labels Printed",
      ...
    }
  ]
}
```

Maintain backwards compatibility: existing callers can use array[0] if needed. Or add `?multi=true` parameter.

**Definition of "active":**
- Not completed/closed/cancelled
- Customer still has open business with us
- Includes: pending approval, in inspection, in service, awaiting parts, ready to ship, recently shipped (until customer confirms received?)

### RC consumer changes (~1.5-2 hrs)

Build ONE accordion component used on both desktop and mobile (responsive via Tailwind):

**Accordion behavior:**
- Renders one row per active estimate
- Most recent (highest #) starts expanded
- Click/tap a collapsed header → expands it, collapses the currently expanded one
- Chevron icon: ▼ when expanded, ▷ when collapsed
- Single job: no chevron, no collapse interaction (just renders the content)
- Hidden entirely when 0 active jobs (shows fallback "Pending update")

**Reuse:**
- Existing `<ProgressBadge>` component renders inside each expanded panel
- Layout component just wraps it with the accordion controls
- Same component used in:
  - Mobile portal Progress tab
  - Desktop portal Progress sidebar
  - CRM conversation panel

**Header indicator:**
- When estimates.length > 1: show count "X active jobs" next to customer name
- When estimates.length === 1: no count (current behavior)
- When estimates.length === 0: no count, fallback content

## Acceptance

### Both desktop and mobile (same accordion)
1. Customer with 1 active job → expanded by default, no chevron interaction
2. Customer with 2+ active jobs → accordion with most recent expanded
3. Click/tap collapsed header → expands it, previously open collapses
4. Only one expanded at a time
5. Most recent estimate # appears first (top)
6. Each item labeled with estimate # only
7. Header shows count: "Drew Hays · 2 active jobs" (when >1)
8. Empty state: "Pending update" fallback (no accordion)

### Consistency
1. Same component used on mobile portal, desktop portal, CRM panel
2. Responsive styling via Tailwind (no separate mobile/desktop versions)
3. No regression for single-job customers (renders same as today)

### Performance
1. Single query returns all active estimates (no N+1)
2. Backwards compat: existing single-result callers still work

## Edge cases

1. **Customer with 5+ active jobs** — both UIs scale (long dropdown / long accordion)
2. **Customer with 0 active jobs** — show "Pending update" placeholder
3. **Closed/cancelled estimates** — not shown
4. **Pending estimates not yet sent** — shown as "active" (staff sees the work)
5. **Multi-watch group approvals** — one estimate, count as one job
6. **Concurrent updates** — refetch on conversation reload, accept eventual consistency

## Out of scope

- Per-estimate action buttons (Send Estimate, Print Label, etc.)
- Detail view for individual jobs from Progress (link to existing estimate page)
- Watch model name in display (estimate # only)
- Notification when other (non-active) job updates
- Customer-side dropdown/filter controls
- Cross-customer aggregate view
- "Most urgent first" sort (defer)

## Considerations for later

- Show watch model below estimate # if customers struggle to identify which is which
- "Most urgent" indicator (job awaiting customer action)
- Aggregate metrics (total active value across all jobs)
- Per-job quick actions
- Compact view toggle for staff with very busy customers

## Related work

- Customer ID architecture (depends on stable RS↔RC linkage)
- Status change notifications (per-job — needs estimate # in notification subject)
- Inspection magic link integration (group approvals may need similar pattern)

## Pickup notes

When resuming:
1. RS endpoint update first (returns array)
2. Maintain backwards compat (existing consumers won't break)
3. Desktop dropdown UI is simpler — ship that first
4. Mobile accordion UI is iterative — ship second
5. Test on Drew Hays (known to have 2 active jobs)

## Build order

1. RS endpoint change (~1 hr) — returns array, backwards compat preserved
2. Single accordion component on RC (~1.5-2 hrs) — responsive for both platforms
3. Header count indicator (~15 min)
4. Testing on Drew Hays (multi-job) + single-job customer (~30 min)

Total: ~3 hours focused build
