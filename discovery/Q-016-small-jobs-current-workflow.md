# Q-016 — Small Jobs Current Workflow (Discovery Spike)

**Type:** Ad-hoc discovery spike (post-Phase-1)
**Priority:** High — informs SPEC-001 workflow-type routing before SPEC weekend
**Owner:** Cursor Agent (autonomous execution)
**Est. duration:** 1-2 hours autonomous work
**Depends on:** Q-001 (Receive Watch), Q-002 (Work Queue state machine), Q-004 (Shop Floor)

---

## Purpose

Map how "small jobs" (single-operation service work like dial installation, permanent link removal, hand adjustment, battery replacement, crown replacement, gasket replacement) are handled in the current Lovable RolliSuite. Determine:
- What workflow steps do small jobs go through today?
- What steps do they skip vs follow vs work around?
- Where does the current system force small jobs through unnecessary rigor?
- Where does the current system fail to capture small-job data properly?
- What ad-hoc handling exists that isn't in the software?

---

## Scope

Files to investigate:
- `apps/rs/src/pages/intake/*` — how small jobs enter the system today
- `apps/rs/src/pages/inspection/*` — do small jobs go through inspection?
- Estimate creation logic — is there any current concept of "small job" vs "full service"?
- Any `is_quick`, `is_small`, `is_walk_in`, or similar flag on jobs/estimates
- Job categorization in `service_categories` and `service_subcategories`
- Pricing tiers — is there a distinct pricing tier for small jobs?
- Turnaround estimation — how is promise date calculated for small jobs?
- QBO invoice generation — do small jobs skip anything in the invoicing flow?
- Shop Floor rendering — do small jobs appear differently from full-service jobs today?

---

## Questions to answer

1. **Categorization today:** Is there any structured concept of "small job" in the current schema, or is it ad-hoc?
2. **Workflow paths taken:** Trace 3-5 example small-job scopes through the current code paths. Do they diverge from full-service somewhere?
3. **Data captured (or not):** For a small job today, what data is captured? What's missing that would be captured for a full-service job?
4. **Skipped steps:** What steps does staff skip manually because they don't apply to small jobs? (Custody transfers, safe placement, inspection photos, etc.)
5. **Custody model:** Do small jobs currently touch the safe, or are they bench-only? How is that tracked (or not tracked)?
6. **Client interaction:** Is walk-in-and-wait supported in the current intake, or does everything assume drop-off?
7. **Concierge concept:** Is there any current concept resembling concierge (staff member owning small-job queue), or is small-job handling distributed across roles?

---

## Output

Save findings to `documentation/discovery/Q-016-small-jobs-current-workflow.md` following the same structure as Q-001 through Q-015 discovery files.

Include:
- Workflow name and description
- Current handling (structured or ad-hoc)
- Example small-job walk-throughs (dial installation, hand adjustment, battery replacement)
- Data gaps identified (what's captured vs what's missing)
- Custody model gaps
- Skipped steps and workarounds
- Concrete proposals for closing each gap in the rebuild
- How findings map to D-024, W-44, and SPEC-001 planning
- Open questions logged in plain language per D-018

---

## Update queue

Update `TASK-QUEUE.md` to include Q-016 as an ad-hoc spike (post-Phase-1). Mark complete when done.

Commit and push per standard cadence. Report SHA.

---

_End of spike packet. This file is the task brief; the investigation results will be written into this same file when executed._
