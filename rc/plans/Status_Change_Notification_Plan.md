# Plan — Status Change Auto-Notification to Customer

**Captured:** Sunday May 17, 2026 (end of long sprint)
**Target build:** This week or Memorial Day weekend
**Estimated effort:** ~3-4 hours cross-system

## The vision

When a job's status changes to one of 7 milestone statuses, the customer automatically receives:
1. A notification message inside their RC portal conversation thread
2. An email with status in subject + warm message + magic link to portal

Result: customers stay informed at key moments, drives portal adoption, reduces "where is my watch?" inquiries.

## Trigger milestones (7 total)

Customer gets notified ONLY at these moments:

1. **Package Received** — "We've got your watch safely"
2. **Inspection Complete (Awaiting Approval)** — action needed from customer
3. **Parts Approved** — work can begin
4. **Work Started (Uncased)** — they're actively servicing
5. **In Testing** — work done, final QC running
6. **Shipped** — on its way back
7. **Delivered** — safely returned

Skipped (too many minor steps): intake labels, disassembly, polishing details, internal department transfers, exit photos as separate milestone (probably bundle into "In Testing" or "Shipped" announcements).

## Message format

### Subject line
Format: "[Status Label] - [Brief context]"

Examples:
- "Package Received - Your Rolex Submariner is here"
- "Inspection Complete - Awaiting Your Approval"
- "Parts Approved - Your watch is now being worked on"
- "Work Started - Your Submariner is on the bench"
- "In Testing - Your Submariner in final QC"
- "Shipped - Your Submariner is on its way home"
- "Delivered - Your Submariner has arrived"

### Body
Short, warm, 1-2 sentences + magic link to portal.

Example for Package Received:
```
Hi Brian,

Your Rolex Submariner has arrived safely at our workshop. 
We'll begin our inspection shortly.

[ View in your portal → ]

— The Rolliworks Team
```

Each milestone gets its own custom warm copy (not generic template).

### Portal-side message
The same message body appears as an "automated message" entry in the customer's conversation thread on RC portal. Visually distinct from staff-typed messages (different background tint or "Automated" badge).

## Architecture

### Where the trigger fires

RW status changes already push to RS via `rollisuite-status-push`. RS has the status timeline. RS already exposes `rc-timeline-status` endpoint that RC polls.

Two options:

**Option A: RC polls and detects changes (no new infrastructure)**
- RC's existing status badge consumer notices a new milestone
- RC's edge function checks "is this a milestone status we haven't notified for?"
- If yes: send the email + insert portal message
- Track "last notified milestone" per conversation

**Option B: RW push directly to RC notification endpoint**
- New webhook from RW → RC notification handler
- Cleaner event flow but requires more cross-system wiring

**Recommendation: Option A** — leverages existing infrastructure, no new cross-system contracts.

### Tracking what's been sent

New table `status_notifications`:
```sql
create table status_notifications (
  id uuid primary key default gen_random_uuid(),
  conversation_id uuid references conversations(id),
  status_step text not null,  -- which milestone status
  notified_at timestamptz default now(),
  unique(conversation_id, status_step)  -- never duplicate
);
```

When a milestone status is detected:
1. Check if `(conversation_id, status_step)` already exists in `status_notifications`
2. If not: send email + portal message, then insert the row
3. If yes: skip (already notified)

## Build phases

### Phase 1: Schema + tracking table (~30 min)
- Create `status_notifications` table
- Add the 7 milestone status names to a constants file

### Phase 2: Detection logic (~1 hr)
- Modify existing status-polling code path
- When a milestone status is detected: check `status_notifications` table
- If new: trigger notification

### Phase 3: Email send (~1 hr)
- Resend-based email
- Custom subject + body per status (7 variants)
- Include magic link to portal
- Match brand styling from existing emails

### Phase 4: Portal message insert (~30 min)
- Insert message row into `messages` table
- Mark as "system" or "automated" type
- Render distinctly in conversation UI
- Triggers the existing "client sees message" UI

### Phase 5: Testing (~30 min)
- Walk a test customer through each milestone
- Verify only 1 notification per milestone per conversation
- Verify email + portal both appear

## Edge cases

1. **Customer has no email** → skip email, send portal message only
2. **Customer has no RC client (rs_customer_id not linked)** → skip both? Or email only?
3. **Status regresses** (e.g., from Shipped back to In Testing — unlikely but possible) → don't send a duplicate
4. **Multiple watches on same estimate** → one notification per job, not per watch (use conversation_id as the dedupe key)
5. **Old conversations** — should we backfill notifications? No — new milestones only going forward

## Out of scope

- Customer-controlled opt-out per milestone (defer to v2)
- Different notification cadence per customer (defer)
- SMS notifications (email + portal only)
- Push notifications (mobile app territory)
- Notification preferences UI for staff

## Considerations for later

- Customer settings: "Notify me at these milestones: ✓ Package received ✓ Shipped ☐ Other"
- Admin override: "Notify customer now" button manually triggers
- A/B test: which message copy drives portal clicks
- Engagement analytics: which milestones get the most clicks

## Pickup notes

When resuming:
1. Confirm Option A architecture (RC polls + detects)
2. Build schema first
3. Detection in same code path as existing status polling
4. Write the 7 custom messages with warm copy
5. Test on Brian or another known test customer

## Related work

- Customer ID architecture (must be stable first — email lookup or UUID linkage)
- Magic link format depends on portal token (verified non-expiring tonight)
- Memorial Day weekend email forwarding (related — customer comms layer)

## Visual: notification status flow

```
RW status change
    ↓
rollisuite-status-push to RS
    ↓
RS updates timeline
    ↓
RC polls rc-timeline-status (existing)
    ↓
RC detects: new milestone status?
    ↓ (yes)
Check status_notifications (already sent?)
    ↓ (no)
Send email via Resend
Insert portal message
Insert status_notifications row
    ↓
Customer sees notification:
  - Email in inbox (subject + magic link)
  - Portal message in conversation thread
```
