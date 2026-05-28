# Email Routing Architecture

**Last updated:** May 27, 2026

RC dissolves the "email silo" by routing customer communication through the system. This document explains inbound and outbound email handling.

---

## Inbound email (customer → RC)

### Domain
- `reply.rolliworks.com` — receives customer replies via Postmark Inbound
- MX records point to Postmark; Postmark POSTs parsed emails to RC's `inbound-email-handler` edge function

### Handler: `inbound-email-handler/index.ts`
Processing order:

1. **Auto-reply filter** — inspect `Auto-Submitted` / `Precedence` headers; skip auto-replies (200 OK, no routing)
2. **Loop prevention** — sender `*@reply.rolliworks.com` → `unmatched_inbound` (`loop_reply_domain`)
3. **Token extraction** — `extractToken()` scans `ToFull` + `CcFull` + `BccFull` for a 16-hex token
4. **Per-customer lookup** — `clients.reply_token = token` → route to most recent non-archived conversation (create new if none)
5. **Per-conversation HMAC fallback** — recompute HMAC for candidate conversations
6. **Staff sender detection** — `team_users.email` match (ilike) or `@rolliworks.com` → `author_type='team'`
7. **Unmatched** — `unmatched_inbound` table for triage

### Known scaling note
The HMAC fallback loads up to 2000 non-archived conversations and recomputes HMAC for each — O(N) brute force. Fine at current scale (~292 conversations), will need indexing at 10k+.

---

## Outbound email (RC → customer)

### Service
- **Resend** for transactional email
- **From:** typically `help@rolliworks.com` (centralized) or `noreply+<token>@quotes.rolliworks.com`
- **Reply-To:** `<token>@reply.rolliworks.com` so replies route back to RC

### Edge function: `portal-send-reply-notification`
- Triggered when staff sends a reply in CRM
- Fetches latest team message
- Sends via Resend with reply token in Reply-To
- Filters out internal messages (defense-in-depth)

---

## Status change notifications (planned, not yet built)

**Architecture decision (locked, Memorial Day weekend build):**
- RW owns email templates (existing mailto infrastructure)
- Replace mailto with Resend auto-send
- RW triggers webhook to RC on status change (including from component lookup)
- RC sends via Resend with customer's reply token + adds message to conversation thread
- Customer reply routes back to RC automatically
- Customer opt-out mechanism
- Each milestone email sent ONCE — backward status moves don't retrigger
- Process map SHOWS backward movement (accurate state) but no email spam

### Milestones (7 + variants)
Package Received, Inspection, Awaiting Approval, Parts Approved, Work Started (Uncased), In Testing, Shipping/Pick Up.
Band flow omits In Testing and Parts; uses Polish/Clean.

### Labels (renamed)
- "Polishing" → "Polish/Clean"
- "Shipped" → "Shipping/Pick Up"

---

## Email forwarding to RC (planned, Memorial Day weekend)

- Forward existing email threads (from Outlook/Gmail) into RC
- Existing customer detection on inbound
- Inbound Inquiries view + Convert to Service Request

---

## Known issues

- **Reply token CC use case** — staff CC a reply token on outbound email; had 3 handler bugs (CC not scanned, staff mis-attributed, no loop prevention), all fixed in code but end-to-end Postmark delivery never confirmed working
- **Postmark dashboard not directly accessible** — no `POSTMARK_API_TOKEN` configured as a project secret; verification requires manual dashboard check
