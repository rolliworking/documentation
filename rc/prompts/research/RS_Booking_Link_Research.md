# Research — Expose Send Booking Link to RC

**→ Paste into: RS chat**

---

# Research mode — no code changes, investigate only

## Context

RS already has a "Send Booking Link" feature today (button/action on customer detail page or similar). It sends a customer-specific booking link with prefilled info.

I want to expose this same functionality to RC so staff can trigger the booking link send from a customer's conversation in RC. Same pattern as Send Portal Invite (RS→RC) but for booking links.

## Investigate

### 1. Find the existing booking link send mechanism

- Where is the "Send Booking Link" button in RS today?
- What edge function or backend code does it call?
- What's the function signature (inputs, outputs)?
- Walk through the full flow: button click → backend call → email send → customer receives link

### 2. Customer-specific prefill logic

- How is the booking link customized per customer?
- What customer fields get embedded (customer_id? email? name? phone?)
- Is the link itself a generated URL, or does it route through RS first?

### 3. Email template

- What email template is used today when sending the booking link?
- What does the customer see? (subject, body, CTA button)
- Is the template editable in RS settings?

### 4. Authentication path

- Today: only RS staff can click the button (gated by role)
- Same `ALLOWED_ROLES` allowlist as `rc-portal-invite`? (We just fixed that to include `manager`)
- For RC integration: should use the same shared-secret pattern (RC service key in header)

### 5. Logging / audit

- Does sending today log to an audit table?
- Does it update anything on the customer record?
- Should RC integration log who triggered it (Mike vs Vienna)?

### 6. Proposed RC integration

**Architecture:**
- New RS edge function `rc-send-booking-link`
- Takes `{customer_id}` or `{customer_email}` from RC
- Auth via `RC_SERVICE_KEY` header (same key already in use)
- Internally calls the existing booking link send logic
- Returns success/failure

OR:

- Reuse existing edge function `rs-send-booking-link` (or whatever name)
- Add option to accept service key auth instead of user JWT
- One function, two callers (RS UI and RC)

Which approach is cleaner given how the current function is structured?

## Specific questions

1. Does the current Send Booking Link button work for ALL customers in RS, or only customers in a certain state?
2. Are there any prerequisites (customer must have email, must have phone, must have past order, etc.)?
3. What happens if you try to send a booking link to a customer who doesn't exist?
4. What's the error handling pattern today?

## Report

- Where the existing function lives (file path)
- Current function signature
- Customer-specific prefill mechanism
- Recommended path to expose to RC (new endpoint vs reuse existing)
- Effort estimate for RC integration

Don't apply changes. Research only.

---

End of prompt.
