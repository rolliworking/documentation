# Memorial Day Weekend Sprint — Email Forwarding to RC

**Sprint window:** Sat May 23 – Mon May 25, 2026 (3-day weekend)
**Estimated effort:** 6-10 hours of focused work, fits comfortably in 3 days with room for other things

## The problem

Customer emails to the shared Exchange inbox are a data silo. Service inquiries get lost in the email pile. Red flag convention (set up today) handles visibility but not the closing-the-loop problem.

## The solution

Forward shared inbox emails to `intake@reply.rolliworks.com`. Postmark Inbound catches them. RC's existing `inbound-email-handler` (extended) processes them into a new "Inbound Inquiries" view with a "Convert to Service Request" button.

## Why this weekend

- 3 days, fewer interruptions
- Cross-system work (Exchange + Postmark + RC)
- Worth doing in focused sprint mode
- Avoids partial implementation getting stale

## Build plan (~6-10 hours)

### Phase A: Email plumbing (~3 hours)

1. **Exchange admin: set up forwarding rule**
   - Shared inbox: when email arrives → forward copy to `intake@reply.rolliworks.com`
   - Verify admin permissions allow external forwarding
   - Test with a sample email

2. **Postmark: configure inbound stream for intake address**
   - Add MX/routing so `intake@reply.rolliworks.com` reaches Postmark
   - Webhook URL points to RC's inbound-email-handler (same as today, with new path/identifier)

3. **RC: extend `inbound-email-handler`**
   - Detect when incoming email was sent to `intake@reply.rolliworks.com`
   - Parse forwarded email headers to extract original sender
   - Store in `unmatched_inbound` with `source = 'forwarded_inbound'`
   - Don't try to match to existing conversation (these are new inquiries by definition)

### Phase B: UI for review (~2-3 hours)

1. **RC: new "Inbound Inquiries" tab/view in CRM**
   - List of unprocessed inbound inquiries
   - Show: original sender name + email, subject, body preview, received time, attachment count
   - Filter: pending / converted / dismissed
   - Sort by recency

2. **Inquiry detail view**
   - Full email body
   - Attachments (especially photos)
   - Action buttons: Convert / Dismiss / Mark Reviewed

### Phase C: Conversion to service request (~1-2 hours)

1. **"Convert to Service Request" button on each inquiry**
   - Click → opens RS estimate page in new tab
   - Pre-fills via URL params: email, name (from sender), description (email body)
   - Status of inquiry changes to "converted" with link to created RS lead

2. **RS-side: accept additional URL params for pre-fill**
   - `?email=<email>&name=<name>&description=<body>`
   - Existing `?customer_id=` and `?lead_id=` params still work
   - Surgical addition to existing estimate page

### Phase D: Edge cases and polish (~1-2 hours)

- Handle duplicate forwarded emails (same email forwarded twice)
- Handle replies to forwarded inquiries (route back to created conversation)
- Loop prevention (forwarded email going back through forwarding rule)
- Vendor / spam filtering (option to mark inquiries as not-customer)

## Gotchas to research before building

1. **Forwarded email format in Postmark**
   - When Exchange forwards, the "From" header changes to the forwarding mailbox
   - Original sender info is in the body or in special "X-Forwarded-From" headers
   - Postmark Inbound has parsing for this — verify capability

2. **External forwarding policy**
   - Some Exchange admin configurations block forwarding to external domains
   - Check Microsoft 365 admin settings: "Mail flow" → "Anti-spam" → outbound spam policy
   - May need to whitelist reply.rolliworks.com

3. **Privacy considerations**
   - Forwarding all shared inbox mail to Postmark = Postmark sees everything
   - Review Postmark's data retention policy
   - Consider if any emails should NOT forward (HR? Personal?)

4. **`unmatched_inbound` table grows**
   - Today: only failed-to-match replies (small)
   - After this: every email forwarded (potentially hundreds/day)
   - Pagination + cleanup of processed inquiries matters

## Acceptance criteria

1. Exchange rule forwards every shared inbox email to intake@reply.rolliworks.com
2. RC's inbox-email-handler stores forwarded emails in unmatched_inbound
3. RC's CRM has an "Inbound Inquiries" view showing forwarded emails
4. Click "Convert to Service Request" → opens RS estimate with pre-filled email/name/description
5. Inquiry shows status: pending / converted / dismissed
6. Photos in forwarded emails are accessible (attachments preserved)

## Existing-customer handling (added to weekend scope)

When a forwarded email arrives, the sender's email may already match:
- A client in RC's clients table (returning customer who used Wix or portal before)
- A customer in RS's customers table (existing customer who never used RC)
- Both (full match)
- Neither (genuinely new)

The Inbound Inquiries flow needs to handle all four cases.

Build:
- On inbound-email-handler: lookup sender email in BOTH RS and RC
- Tag the inbound_inquiry row with `match_status`: rc_match | rs_match | both_match | none
- Display tag in Inbound Inquiries view ("Returning client: Sarah Chen, last serviced 2024" vs "New inquiry")
- "Convert to Service Request" routes differently per tag:
  - rc_match: add new conversation to existing client, prefill from their data
  - rs_match (not in RC): create RC client linked to existing RS customer_id, then conversation
  - both_match: new conversation on existing client
  - none: standard new customer flow

This is ~1-2 hours added to the weekend scope. WORK THROUGH THIS WEEKEND.

## Out of scope for this weekend

- AI extraction of watch reference / service type (Phase 3)
- Auto-classification (inquiry vs vendor vs spam)
- Microsoft Graph API direct integration (rejected as too expensive)
- Outlook plugin / add-in
- Replacing Exchange as primary email tool

## Pre-sprint prep (do during the week before)

- Confirm Exchange admin permissions for external forwarding
- Pick a name for the intake address (intake@reply.rolliworks.com or similar)
- Decide forwarding scope: ALL shared inbox emails, or only red-flagged ones?
  - If only red-flagged → richer signal but requires staff to manually flag first
  - If ALL → captures everything, more noise in inbound view

## After this weekend ships

- Vienna and Mike use the new "Inbound Inquiries" view as their service inquiry queue
- Red flag in Outlook can be deprecated OR kept as a backup visual signal
- Email silo problem largely solved
- Foundation for Phase 3 AI extraction features

## Status going into the weekend

Hopefully by Friday May 22:
- Today's backfill complete and verified
- Create Estimate button confirmed working across multiple customers
- Progress badges (this week's planned ship) live
- Any other small improvements complete
- This sprint starts from a stable foundation

## Notes

If time pressure during the weekend, ship Phases A + B (gets emails into RC + visible). Phase C (conversion button) can wait if needed — staff can still see the queue and manually create in RS.

The CONVERSION automation is the icing. The CAPTURE (Phase A) is the meat.
