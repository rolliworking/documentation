# Research — Inspection Notes / Approval Flow

**→ Paste into: RW chat**

---

# Research mode — no code changes, investigate only

## Context

Today RW sends clients an inspection email with a link to an HTML inspection page where they opt into a service agreement. I want to understand the full flow before designing how RC's client portal can integrate.

Specifically: clients should be able to respond to inspection notes inside RC's portal instead of (or in addition to) the existing standalone HTML page. The existing inspection email should ideally also include a magic-link to the RC portal so clients can use whichever they prefer.

## Investigate

### 1. The current inspection email flow

Walk through the complete sequence from technician completing inspection to client receiving email:

- What triggers the inspection email send? (Status change? Manual click? Button in UI?)
- What edge function sends it?
- What's in the email — full inspection notes? Just a link?
- What's the URL the client lands on?
- Is the URL gated by anything (token? expiry? login?)
- What's the page they land on (file references)?

### 2. The HTML inspection page itself

Document the page client sees:

- What inspection data is rendered (notes, photos, estimated cost, parts list)?
- What can client DO on this page? (approve, decline, comment, request changes)
- Is there a comment/response field? Or just approve/decline buttons?
- After clicking approve, what happens? (DB write, follow-up email, status transition)
- After clicking decline, what happens?
- Is there a way for client to come BACK to the page later? (re-open via same link)

### 3. Service agreement / terms acceptance

- Is the service agreement embedded on the inspection page, or separate?
- Is it acknowledged by checkbox? Explicit signature? Just by clicking approve?
- Where is the acceptance recorded (table, field)?
- Does it need to be re-accepted for each service?

### 4. Client response capture

When client approves or comments:
- Where does the data land (table, columns)?
- Is there a "client_responses" or similar table?
- How does staff get notified of the response?
- Can client UPDATE their response, or is it final?

### 5. Tokens, authentication, expiry

- What's the URL structure for the inspection link (token, params)?
- How is access controlled?
- Token expiry / one-time use?
- Anti-tampering (signed URLs)?

### 6. What can be exposed to RC

For RC to surface inspection notes inside the portal:
- Is the inspection data accessible via existing endpoints/APIs?
- Could RC display the same notes, photos, estimate alongside the portal conversation?
- Could RC trigger the same approval action (proxying through to RW)?
- What's the simplest way to expose inspection data to another consumer (RC)?

### 7. Magic link integration possibility

For the email to include BOTH the standalone HTML page link AND a magic link to RC portal:
- What does RW know about a client's RC presence?
- Does RW have access to the client's `rc_client_token`?
- Would RW need to call RC to get one, or query a shared store?
- Or should RC be the email sender (not RW), so RC has full context to include both links?

### 8. Workflow considerations

If client responds via the RC portal (vs the standalone page):
- How does RW know about it?
- Does RW need to be informed, or just RS?
- Race conditions if client clicks both (HTML page AND portal)?

## Report

Write a structured findings doc with:
- Inspection email send flow (trigger → recipient)
- Inspection page UX inventory (what client sees + can do)
- Service agreement / terms storage shape
- Response capture mechanism
- Token/auth scheme
- Existing data exposure paths for external consumers
- Recommended approach for RC integration (high-level architecture, not detailed code)
- Concerns / edge cases

Don't apply changes. Research only.

---

End of prompt.
