# Research — RS Process Flow + Shipping/Pickup States Relationship with RW

**→ Paste into: RS chat**

---

# Research mode — no code changes, investigate only

## Context

We're building a customer-facing Progress display that needs to show accurate workflow status. We thought RW (workshop) owned the flow, but RS has "Shipped" and "Picked Up" states AND its own process flow. Need to understand the relationship before building anywhere else.

## What I'm trying to figure out

The customer-facing portal Progress display currently shows steps like:
- Package Received → Intake Labels → Inspection Photos → Awaiting Approval → Work Started → In Testing → Shipping/Pick Up → Delivered

We need to understand: which of these states live in RS, which in RW, and how they connect.

## Investigate

### 1. RS process flow states

- What states does RS track for an estimate/order?
- Specifically: where do "Shipped" and "Picked Up" live? (estimate status? separate table?)
- List all RS workflow states in order
- Which states are RS-owned vs synced from RW?

### 2. RW process flow states

- RW has `jobs.status` with values: intake, inspection, waiting_approval, in_queue, uncased, in_progress, parts_approval, parts_on_order, in_testing, waiting_components, finished
- RW has `jobs.flow` with: STANDARD_FLOW, SPLIT_FLOW, BAND_ONLY_FLOW
- After RW marks a job "finished" — what happens in RS? Does RS take over for shipping/pickup?

### 3. The handoff

- When does a job transition from "RW owns the state" to "RS owns the state"?
- Is it: RW handles all workshop work → RS handles all shipping/delivery?
- Or: RW handles everything until "finished," then RS records "Shipped" / "Picked Up" separately?
- What's the data flow between them?

### 4. Existing RS process flow page

- You said RS has a "process flow" — is this a customer-facing UI? Internal staff UI? Both?
- Where does it live in RS code?
- What states does it visualize?
- Is this where "Shipped" / "Picked Up" is set or just displayed?

### 5. Source of truth for each state

For each customer-facing step, tell me which system owns the truth:

| Customer-facing step | Owner (RS or RW?) | Field/table |
|---|---|---|
| Package Received | ? | ? |
| Intake Labels | ? | ? |
| Inspection Photos | ? | ? |
| Awaiting Approval | ? | ? |
| Parts Approved | ? | ? |
| Work Started | ? | ? |
| In Testing | ? | ? |
| Shipped | ? (probably RS) | ? |
| Picked Up | ? (probably RS) | ? |
| Delivered | ? | ? |

### 6. The relationship architecture

- Does RS push state changes to RW, or RW push to RS?
- Is there a single "current customer status" anywhere, or does RC have to assemble it from both systems?
- Is the current `timeline-status` endpoint on RS or RW? Which system has the "calculate current customer-facing step" logic?

## Specific questions

1. Should the band flow split logic live in RS or RW? (Now that we know RS has flow logic too)
2. For "Shipping" vs "Pick Up" customer preference — is that captured anywhere? Where?
3. If a watch is "Shipped" in RS, does RW's jobs.status reflect that, or does it stay at "finished"?
4. If a customer picks up in person, what triggers the "Picked Up" state in RS?

## Don't apply changes

Just diagnose the architecture. Report back so we can decide where the band flow split (Task 1) and timeline endpoint logic should actually live.

## Output wanted

1. Diagram (text or markdown) showing: estimate created → workshop work → shipping/pickup → delivered, marking which system owns each phase
2. Recommendation: where should band flow split logic live (RW or RS)?
3. Recommendation: where should the customer-facing "current step" be computed?
4. Any gotchas or messy edge cases between the two systems

---

End of prompt.
