# Research — RC Inbound Email Handling & Plus Addressing

**→ Paste into: RC chat**

---

# Research mode — no code changes, investigate only

## Context

We want to embed a per-client routing email address (like `reply+CLIENT_TOKEN@reply.rolliworks.com`) in the CC field of inspection approval emails sent from RW. When the customer replies to the email, the CC address would route their reply into their RC conversation automatically.

Before building, need to understand current state of RC's inbound email infrastructure.

## Investigate

### 1. Current inbound email handler

- Where does it live? (`supabase/functions/inbound-email-handler/index.ts`?)
- What addresses does it currently handle?
- Walk through the routing logic — how does it decide which conversation an inbound email belongs to?

### 2. Postmark Inbound configuration

- What domain is configured? (`reply.rolliworks.com`?)
- Is the entire `*@reply.rolliworks.com` routed to the handler, or specific addresses only?
- Are there any wildcards or pattern matching in place?

### 3. Plus addressing support

When an email arrives at `reply+TOKEN@reply.rolliworks.com`:
- Does Postmark deliver it to the handler at all? (sometimes mail servers reject `+` addresses)
- Does the handler extract `TOKEN` from the local part?
- Or does it just see the full `reply+TOKEN` and match against exact addresses?

### 4. Current reply token mechanism

We use HMAC reply-to tokens today for routing replies. How does this differ from what we'd want for the proposed CC routing:

- Are tokens embedded in From or Reply-To addresses?
- HMAC signed, or just opaque?
- Tied to a conversation, or a client?

### 5. Customer-to-client matching

For the proposed flow, the CC address needs to identify the CLIENT (not a specific conversation). When a reply arrives:
- We need to know WHICH client sent it (via the `+TOKEN` part)
- We need to figure out which CONVERSATION it goes into (most recent active? Or always create a new one?)

Walk through what would need to change in the handler to support client-scoped routing (vs the current conversation-scoped routing).

### 6. Magic link / client token plumbing

- What stores the client_token? (Probably `clients.token` or similar)
- Can a routing email be deterministically constructed from a client_token?
- Or do we need a separate field like `clients.reply_email`?

### 7. Security considerations

- If tokens are exposed in CC headers, they could be forwarded/leaked
- Mitigations available (signed tokens, expiry, etc.)
- Current pattern for the existing magic link tokens — same security level acceptable?

## Specifically for the proposed feature

The flow we want to support:

```
RW builds inspection email
    ↓
RW calls RC endpoint: "give me routing email + portal URL for customer X"
    ↓
RC returns: {
  routing_email: "reply+CLIENTTOKEN@reply.rolliworks.com",
  portal_url: "https://my.rolliworks.com/client/TOKEN/c/CONVID"
}
    ↓
RW substitutes both into email template
    ↓
Technician sends via mailto
    ↓
Customer replies → reply+CLIENTTOKEN@reply.rolliworks.com
    ↓
Postmark Inbound → RC handler
    ↓
Handler extracts CLIENTTOKEN, finds client's conversation, adds reply to thread
```

What changes in RC's stack to make this work? List them.

## Report

Write findings with:
- Current inbound email handler architecture
- Postmark Inbound capabilities and limits (especially around `+` aliasing)
- Token model assessment
- Specific changes needed to support the new flow
- Time estimate for each change
- Security/operational concerns

Don't apply changes. Research only.

---

End of prompt.
