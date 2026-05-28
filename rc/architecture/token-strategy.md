# Token Strategy

**Last updated:** May 27, 2026

RC uses several token types for different purposes. Understanding their scopes and blast radius is important for security.

---

## Token types

### 1. Portal magic link token (`clients.token`)
- **Purpose:** Grants customer access to their portal
- **Scope:** FULL portal access — read all their conversations, view photos, post messages as the client
- **Format:** Random string, stored on `clients.token`
- **Lifetime:** Permanent (does not expire) — verified
- **URL form:** `https://my.rolliworks.com/client/<token>/c/<conversation_id>`
- **Blast radius if leaked:** HIGH — full portal session for that customer
- **Generated:** On client creation (same pattern as reply_token)

### 2. Per-conversation HMAC reply token
- **Purpose:** Routes inbound email replies to the correct conversation
- **Scope:** Identifies one conversation
- **Format:** HMAC-SHA256 of `conversation_id` with `RC_INBOUND_HMAC_SECRET`, hex, truncated to 16 chars
- **Generation:** `_shared/inbound-token.ts` → `conversationInboundToken(conversationId)`
- **Deterministic:** Same conversation → same token forever
- **Used in:** Outbound email `Reply-To: <token>@reply.rolliworks.com` and `From: noreply+<token>@quotes.rolliworks.com`
- **Storage:** None — purely computed
- **Blast radius if leaked:** LOW — post one message to one conversation

### 3. Per-customer reply token (`clients.reply_token`)
- **Purpose:** Permanent address customers/staff can use to route email into RC
- **Scope:** Identifies one customer, routes to their most recent active conversation
- **Format:** Random 16 hex (64 bits), UNIQUE indexed
- **Lifetime:** Permanent, rotatable
- **URL form:** `<reply_token>@reply.rolliworks.com`
- **Display:** Shown in CRM conversation header under client name (staff-only) with Copy button
- **Blast radius if leaked:** LOW — post one message to that customer's conversation
- **Backfill:** All 271 existing clients backfilled with unique tokens

### 4. Service keys (`RC_SERVICE_KEY`, etc.)
- **Purpose:** Cross-system authentication (RS↔RC endpoints)
- **Scope:** Server-to-server only
- **Used by:** `rc-portal-invite`, `get-rc-portal-link`, inspection approval push
- **Must NOT** appear in URLs, web responses, or client-side code

---

## Inbound email token decode order

When an inbound email arrives at `reply.rolliworks.com`, the `inbound-email-handler` resolves the token in this order:

1. **Auto-reply filter** — check `Auto-Submitted` / `Precedence` headers; skip if auto-reply (out-of-office, vacation)
2. **Loop prevention** — if sender is `*@reply.rolliworks.com`, drop to `unmatched_inbound` with reason `loop_reply_domain`
3. **Per-customer token lookup** — match `clients.reply_token` (indexed point query, fast)
   - If found → route to client's most recent non-archived conversation, or create new
4. **Per-conversation HMAC scan** — recompute HMAC for candidate conversations (O(N) brute force, fine at current scale)
5. **Unmatched** — drop to `unmatched_inbound` for triage

Token extraction reads `ToFull` + `CcFull` + `BccFull` (so CC'd tokens route correctly).

---

## Staff sender detection

When an inbound email's sender matches a `team_users.email` (case-insensitive) OR ends with `@rolliworks.com`:
- Message attributed as `author_type='team'` (not `client`)
- Display name pulled from `team_users.name`
- Conversation state/unread NOT bumped for team authors

This enables staff to CC a reply token onto outbound emails and have the thread route into RC correctly attributed.

---

## Security recommendations

- Keep portal token (`clients.token`) and reply token (`clients.reply_token`) SEPARATE — different lifetimes, different leak surfaces. Do NOT unify them.
- Reply tokens are routing identifiers, not bearer credentials — combined with `inbound_sender_mismatch` flag, leaked tokens have limited impact.
- For rotation: regenerate `reply_token`; historical per-conversation HMAC tokens still resolve via the HMAC scan path.
- Consider auto-reply filtering to prevent OOO/vacation emails from routing into RC forever once a token is in a customer's address book.
