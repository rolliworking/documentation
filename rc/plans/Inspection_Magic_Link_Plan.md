# Plan — Inspection Email Magic Link Integration

**Captured:** Sunday May 17, 2026 (evening, end of long sprint)
**Target build:** This week
**Estimated effort:** ~6-10 hours cross-system

## The vision

Embed RC portal routing into every touchpoint of the inspection approval flow. Three reinforcement points:

1. **CC field** of the inspection email contains `reply+CLIENTTOKEN@reply.rolliworks.com` — when customer replies, it lands in their RC conversation automatically
2. **Body of the email** contains the magic link URL alongside the existing approval URL
3. **Bottom of the HTML approval page** has the magic link with verbiage like "Have comments or requests? Reply here: {magic_link}"

End result: customer has three doors into RC, every interaction reinforces the portal exists.

## Why this works

- Mailto flow preserved — technician still reviews email before sending (intentional QA)
- Existing approval URL flow unchanged
- Customer doesn't need to "know" about magic links — they just hit Reply and end up in the portal
- Drives adoption without forcing it

## Architecture

```
RW inspection report
    ↓ (technician clicks "Send Approval")
RW calls new RC endpoint: GET /rc-routing-info?customer_id=...
    ↓
RC returns:
  - routing_email: reply+CLIENTTOKEN@reply.rolliworks.com
  - portal_url: https://my.rolliworks.com/client/TOKEN/c/CONVID
    ↓
RW substitutes into email template (To, CC, Body) and HTML approval page
    ↓
mailto: opens with everything filled in
    ↓
Technician reviews, hits Send
    ↓
Customer receives email
    ↓
Customer replies → goes to CC address (reply+TOKEN@)
    ↓
Postmark Inbound → RC inbound-email-handler
    ↓
Handler extracts TOKEN, finds client's most recent active conversation
    ↓
Reply appears in their RC conversation
```

## What needs to change

### RC side

1. **New endpoint `rc-routing-info`** that returns routing_email + portal_url given customer_id or email
2. **Extend inbound-email-handler** to support `+TOKEN` aliasing — extract token, route to that client's conversation
3. **Postmark Inbound config check** — verify `+` addresses route to RC handler
4. **Decide conversation routing logic**:
   - Reply to existing active conversation?
   - Create new conversation if none active?
   - What if multiple active conversations?

### RW side

1. **Update inspection email template** in `email_templates` table — add `{{rc_portal_url}}` placeholder
2. **Modify "Send Approval" handler** in `InspectionReport.tsx` to call RC endpoint and substitute values
3. **Add CC field to mailto URL construction** — include routing_email
4. **Update HTML approval page** (`ApproveInspection.tsx`) — add magic link section at bottom

### RS side

Likely no changes. Could be involved if customer lookup goes through RS first (depends on whether RW knows the customer's email/RS customer_id at email send time).

## Open questions (need answers before building)

1. **Postmark plus-addressing support** — does it actually work with current config?
2. **Token format** — reuse existing `rc_client_token` or new field?
3. **Multiple active conversations** per client — which one does the reply route to?
4. **Token security** — is leaking `rc_client_token` in CC headers acceptable? Or do we need a different per-message token?
5. **Service agreement** — when customer responds via portal (not approval page), what happens to T&C acceptance? Does it still require the approval page submission?

## Constraints / preserved behaviors

- ✅ Keep mailto flow (technician QA preview)
- ✅ Keep existing approval URL working
- ✅ Don't break the standalone HTML approval page
- ✅ T&C acceptance still requires the explicit form submission

## Phased build

### Phase 1: RC endpoint + minor inbound handler updates (~2-3 hrs)
- Build `rc-routing-info` endpoint
- Extend inbound handler for `+TOKEN` aliasing
- Test manually with sample emails

### Phase 2: RW email template + mailto modification (~2-3 hrs)
- Update template to include both URLs
- Modify mailto builder to add CC + new body content
- Test with technician walkthrough

### Phase 3: HTML approval page footer (~1 hr)
- Add magic link section to bottom of approval page
- Pull magic link URL from approval data

### Phase 4: Testing + iteration (~1-2 hrs)
- End-to-end test on a real customer
- Verify reply routing works
- Catch edge cases

## Out of scope (for this build)

- Better mailto preview/send pipeline (separate Phase 3 work)
- Server-side email sending via Resend (revival of dead code, separate work)
- Email analytics / open tracking
- Forwarding detection / token rotation
- HTML approval page hosted in RC (just adding link to existing RW page)

## Dependencies / blockers

- RC research findings (pending — research prompt drafted)
- Postmark config check (likely OK, needs verification)
- Confirmation on multiple-active-conversations routing logic

## Notes from tonight's discussion

- Technician QA via mailto is intentional, not a workaround
- Three touchpoints (CC + body + approval page) are deliberate adoption reinforcement
- Goal is customer-driven portal adoption, not forced migration
- High-value customers ($30K jobs) deserve premium portal experience
- **Visual hierarchy: approval URL and magic link should be EQUALLY prominent** — split the visual hierarchy, both above the fold. Customer picks their preferred path; we don't bias them either way.

## Pickup notes

When resuming this work:
1. Get RC research findings on inbound email handling
2. Confirm Postmark plus-addressing capability
3. Decide multiple-conversation routing logic
4. Then start Phase 1 build

## Related work in flight

- Memorial Day weekend: email forwarding + existing-customer handling
- This sprint: this magic link integration
- Both touch RC's inbound-email-handler — coordinate to avoid stomping on each other's changes
