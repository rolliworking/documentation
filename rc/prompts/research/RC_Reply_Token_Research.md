# Research — Per-Customer Reply Token for Display in CRM

**→ Paste into: RC chat**

---

# Research mode — no code changes, investigate only

## Context

When RC sends an outbound email to a client, the Reply-To header includes a unique token like `reply+TOKEN@reply.rolliworks.com`. Postmark Inbound receives the client's reply, decodes the token, and routes back to the right conversation.

Today this token is per-message (or per-thread) and embedded in outbound email metadata. It's NOT displayed in the CRM UI anywhere.

## What I want to build

A permanent per-customer reply token, displayed in the CRM conversation panel under the customer's name. Staff can copy it and paste into Outlook/Gmail when replying to old email threads outside RC. Customer's reply lands in their RC conversation automatically.

```
Drew Hays · Rolex Movement Service
drew.hays1@gmail.com · 5126362936
📋 reply+CUSTTOKEN@reply.rolliworks.com  [Copy]   ← NEW
```

## Investigate the current state

### 1. Outbound reply-to token generation

- Where does the Reply-To token get generated when RC sends an outbound email?
- File path + function
- Token format (HMAC? Random string? Includes conversation_id or message_id?)
- Per-message vs per-thread vs per-customer scope

### 2. Postmark Inbound handler

- Where does it decode the token?
- What does it map to (conversation_id? user? customer?)
- Is the routing strict (token must match exactly) or flexible?
- Can it support multiple token formats simultaneously?

### 3. Token storage

- Are reply tokens stored anywhere in the database, or only computed/embedded in emails?
- If stored: which table, what columns?
- If computed: from what data?

### 4. clients.token vs reply tokens

- The `clients.token` (portal magic link) we verified doesn't expire
- Could we use the SAME token as the reply token, or do they need to be separate?
- Pros/cons of unified vs separate tokens

### 5. Edge cases

- What happens if a customer replies to an OLD email with an old token?
- Token rotation / revocation possible today?
- Multiple emails from same customer → same token or different?

## Proposed architecture (Option A from spec)

Add a NEW permanent per-customer reply token:

**Schema:**
```sql
alter table clients add column reply_token text unique;
-- Generated on client creation (similar to clients.token)
-- Stays the same forever
```

**Inbound handler update:**
- Accept BOTH per-message tokens (existing) AND per-customer tokens (new)
- Per-customer token → look up client by reply_token → route to their most recent active conversation (or create one)

**UI:**
- Display in conversation panel header under customer name
- Format: `reply+REPLYTOKEN@reply.rolliworks.com`
- Copy button next to it

**Backfill:**
- For all 259 existing clients, generate a reply_token
- One-time migration

## Specific questions

1. **Is the Reply-To token currently per-message or per-conversation?** If per-conversation, we might be able to reuse infrastructure. If per-message, we definitely need a new permanent token.

2. **Can the Postmark Inbound handler accept multiple token formats?** Adding a permanent token MUST NOT break existing per-message routing.

3. **What's the format/length of the current tokens?** New per-customer tokens should match the security/usability characteristics.

4. **Could `clients.token` (the portal magic link) double as the reply token?** This would be elegant — one token per customer covers both portal access AND email routing. But might have security implications worth considering.

5. **Where would the "most recent active conversation" mapping fail?**
   - Customer with multiple conversations → which conversation receives the reply?
   - Customer with no conversations → create new one?
   - Customer with archived conversation → unarchive on reply?

## Report

- Current token system architecture (file paths, scope, storage)
- Recommendation for permanent per-customer tokens (new column vs reuse clients.token)
- How to update Postmark Inbound to accept both formats safely
- Time estimate for the full feature (schema + backfill + handler + UI)
- Any security concerns
- Anything I haven't considered

Don't apply changes. Research only.

---

End of prompt.
