# Build — "Push to RC" Button on Customer Detail Page

**→ Paste into: RS chat**

---

# Build mode — Push existing customers to RC manually

## Surgical fix only

Touch ONLY the files needed for this specific change. Don't refactor unrelated code. Don't add "while we're here" improvements. Minimal blast radius. No scope creep beyond what's requested. If you spot other issues during this work, flag them at the end of your report — don't fix them.

## Context

The existing Wix → RS → RC pipeline (via `wix-intake-webhook` calling `RC_PORTAL_INTAKE_URL`) only handles NEW Wix intake leads from ~April 27, 2026 onward. Historical customers (~1,150 created before the bridge wired up) have no RC presence.

We need a way for staff to manually push individual customers from RS to RC. This will be used selectively when:
- A historical customer reaches back out (email reply, phone call)
- The Wix → RC push failed (the ~1% with NULL `rc_conversation_id`)
- Staff wants to invite a specific existing customer to the portal

Wix chat has been removed from rolliworks.com. Customers replying to old conversations come back through email; staff identifies them in RS and pushes them to RC manually.

## What to build

### 1. UI — "Send Portal Invite" button on customer detail page

Add a button to the customer detail page in RS. Specific requirements:

- **Label:** "Send Portal Invite" (clearer to staff than "Push to RC")
- **Placement:** Prominent on the customer detail page — top of page or near customer info section
- **State handling:** 
  - Shows "Send Portal Invite" if customer has NULL `rc_client_token`
  - Shows "View in RC" (link to RC's conversation) if customer already has `rc_client_token`
  - Or shows "Re-send Invite" if you decide to support repeated invites
- **Confirmation dialog:** "Send a portal invite to {customer.email}? They'll receive an email with their portal link." [Cancel] [Send Invite]
- **Loading state:** spinner while request in flight
- **Success state:** Toast/banner "Portal invite sent. Conversation created in RC."
- **Error state:** Toast/banner with error details, retry option

### 2. Backend — Endpoint or RPC

Build an RS endpoint (edge function or RPC) that:

- Authenticates the staff user (only admin/team can push)
- Takes a customer_id
- Reads the customer record from `customers` table
- Calls `RC_PORTAL_INTAKE_URL` with the SAME payload shape the Wix intake uses:
  ```
  {
    email: customer.email,
    name: customer.name (split into first/last if needed),
    phone: customer.phone,
    service_type: ??? (see "service_type" decision below),
    initial_description: "Portal invitation sent by team",
    intake_lead_id: <optional reference if customer has any intake_lead>
  }
  ```
- Authorization: `Bearer $RC_SERVICE_ROLE_KEY` (same as Wix intake)
- On success: receive `{ client_token, conversation_id }` from RC
- Write those values back to `customers.rc_client_token` and `customers.rc_conversation_id`
  - Add these columns to `customers` table if they don't exist (similar to how `intake_leads.rc_client_token` works)
- On failure: log the error, return error to UI

### 3. service_type handling

The customer record may not have a clean service_type (Wix intake_leads have it; not sure about manually-created customers).

Options:
- Use the customer's MOST RECENT intake_lead's service_type if available
- Default to "Service Inquiry" if no intake_lead exists
- Allow staff to pick the service_type in a dropdown before sending

Recommend: use most recent intake_lead service_type, fallback to "Service Inquiry", no dropdown for now.

### 4. Welcome email triggering

The push should automatically trigger RC's welcome email to the customer.

RC's portal-intake-router currently fires the welcome email automatically when called from Wix. If that's true, no extra work needed — the welcome email fires from this push too.

Verify: when RC receives the push, does it call portal-send-welcome? If not, the manual push needs to either:
- Fire welcome email itself (call portal-send-welcome after creating conversation)
- Or have RC fire it from the receive endpoint

This is a coordination point with RC. If you're not sure, flag it and we'll ask RC.

### 5. Audit trail

Track who sent invites:
- Add `customers.invite_sent_at` and `customers.invite_sent_by` columns
- Optional: separate `customer_invites` table if we want full history (multiple invite attempts, etc.)

For now, the two columns on customers table are enough.

### 6. Duplicate prevention

Before pushing, check if customer already has `rc_client_token`. If yes:
- Show "Already in RC" instead of inviting again
- Provide link to their RC conversation
- Optionally: "Re-send Invite" button that creates a new welcome email but doesn't create a duplicate conversation

## What NOT to build now

- Bulk push for all 1,150 historical customers (separate decision)
- Wix contacts CSV import (separate flow)
- Wix chat history capture (Wix doesn't expose this)

## Acceptance criteria

1. Customer detail page shows "Send Portal Invite" button
2. Click confirms via dialog, then calls backend
3. Backend pushes to RC, receives token + conversation ID
4. Customer record updated with rc_client_token, rc_conversation_id, invite_sent_at, invite_sent_by
5. Customer receives welcome email from RC with magic link
6. Returning to customer page: button now shows "View in RC" linking to the conversation
7. Errors are caught and shown to staff (network errors, RC errors, etc.)
8. Only admin/team roles can use this button

## Open coordination question for RC

Does RC's `portal-intake-router` currently fire the welcome email automatically when called from RS? If not, we need to coordinate so the manual push also triggers the welcome email. Flag this if you can't determine from RS code alone.

Time estimate: 2-3 hours.

After deploy, I'll verify:
- Find a test customer (Brian Poulson is a real candidate)
- Click Send Portal Invite
- Confirm email arrives
- Click magic link, verify portal access
- Confirm customer record now shows linked

---

End of prompt.
