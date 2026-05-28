# Build — Band Flow Split for Customer Progress Display

**→ Paste into: RW chat**

---

## Surgical fix only

Touch ONLY the customer-facing status flow logic. No other changes.

## Goal

Customer Progress badge on RC portal currently shows watch-service steps for ALL jobs, including band-only jobs. Customers with band-only service see "In Testing" which doesn't apply. This is customer-facing and embarrassing.

Fix: differentiate band vs watch flow at the endpoint level so RC displays the correct progression for each.

## What you already have on shopfloor

You differentiate band vs watch flow internally already. Same logic needs to flow through to the customer-facing endpoint.

## Required investigation (~10 min, then build)

### Step 1: Identify the differentiator
- What field determines band vs watch? (`inspection.job_type`? `department_tag`?)
- What value indicates "band only"? (e.g., `department_tag` contains "B"?)
- Show me the specific field + values

### Step 2: List the band-only flow steps
Confirm the actual band flow steps in order. My best guess:
```
Package Received → Intake Labels → Inspection Photos → Awaiting Approval
→ Polish/Clean → Shipping/Pick Up → Delivered
```

vs watch flow:
```
Package Received → Intake Labels → Inspection Photos → Awaiting Approval
→ Parts Approved → Work Started → In Testing → Shipping/Pick Up → Delivered
```

Notable differences:
- Band has NO "In Testing"
- Band uses "Polish/Clean" (rename from "Polishing")
- Both use "Shipping/Pick Up" (rename from "Shipped")

Confirm or correct the band flow steps before building.

### Step 3: Build the fix
Update the `job-status` endpoint to:
1. Detect job type (band vs watch)
2. Return appropriate flow steps for each
3. Compute current_step and next_step within the correct flow

**Add to response:**
```json
{
  "flow_type": "watch" | "band",
  "current_step": "...",
  "current_step_label": "...",
  "completed_steps": [...],
  "next_step": "...",
  "next_step_label": "..."
}
```

For band jobs:
- "In Testing" never appears in any field
- Use band-flow next_step computation

For watch jobs:
- Full watch flow unchanged

## Also in this commit (small label changes)

While in the status label code, update these labels:
- "Polishing" → "Polish/Clean"
- "Shipped" → "Shipping/Pick Up"

These labels apply to both flows.

## Acceptance

1. Endpoint returns `flow_type` correctly for band vs watch
2. For band jobs: "In Testing" never appears in any response field
3. For watch jobs: full flow including "In Testing" still works
4. "Polish/Clean" label used instead of "Polishing"
5. "Shipping/Pick Up" label used instead of "Shipped"

## Test

1. Pick a band-only job in RW (any recent one)
2. Call `job-status` endpoint with that job's data
3. Verify response: `flow_type: "band"`, no "In Testing" anywhere, "Polish/Clean" appears in steps
4. Pick a watch job
5. Verify response: `flow_type: "watch"`, full flow with "In Testing", "Polish/Clean" label, "Shipping/Pick Up" label

## Out of scope

- RC display changes (RC just renders what you return)
- New endpoints (modify existing job-status only)
- Customer notifications (separate sprint)
- RS→RW data sync issues (separate work)

## Workflow

1. Investigate Step 1 + Step 2
2. Report back what you found (band field + steps list)
3. Wait for my confirmation
4. Then build Step 3

Don't apply code changes until I confirm the band flow steps look right.

---

End of prompt.
