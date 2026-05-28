# Build — Per-Customer Reply Token

**→ Paste into: RC chat**

---

## Surgical fix only

Touch ONLY the files listed below. Don't refactor unrelated code. Don't add "while we're here" improvements.

## Decisions locked from research

| Decision | Value |
|---|---|
| Token generation | Random 16 hex (UNIQUE constraint, indexed) |
| Token storage | New column `clients.reply_token` |
| Inbound lookup order | Per-customer token first (indexed), then existing HMAC fallback, then unmatched |
| Auto-reply filter | Yes — check Postmark `Auto-Submitted` header, skip routing if present |
| No active conversations | Create new conversation (route email into it) |
| Display format | `TOKEN@reply.rolliworks.com` (bare, no `+`) |
| Display location | Conversation panel header, under client name + email |
| UI element | Address text + Copy button + toast confirmation |

## Build phases

### Phase 1: Schema + backfill (~30 min)

```sql
-- Add the column
alter table clients add column reply_token text;
create unique index idx_clients_reply_token on clients(reply_token) where reply_token is not null;

-- Backfill all existing clients with random 16-hex tokens
-- Use crypto.randomBytes or similar — must be cryptographically random
update clients 
set reply_token = encode(gen_random_bytes(8), 'hex')
where reply_token is null;

-- Make it NOT NULL going forward
alter table clients alter column reply_token set not null;

-- Default for new inserts (generated in app code or DB trigger)
-- Recommend: generate in app code where clients are created
```

For all 259 existing clients, generate tokens. Verify uniqueness post-backfill.

For NEW client creation (`portal-intake-router`), generate the token at insert time. Same pattern as `clients.token` is generated today.

### Phase 2: Inbound handler update (~30 min)

In `supabase/functions/inbound-email-handler/index.ts`:

**Auto-reply filter (NEW — add at top of handler):**
```ts
const autoSubmitted = headers["auto-submitted"] || headers["Auto-Submitted"];
if (autoSubmitted && autoSubmitted.toLowerCase() !== "no") {
  // Log and skip routing
  console.log("Skipping auto-reply email:", autoSubmitted);
  return new Response(JSON.stringify({ ok: true, skipped: "auto-reply" }), { status: 200 });
}
```

**New routing path (PRIORITY 1):**
Before the existing HMAC scan, try per-customer token lookup:

```ts
// After extractToken() returns a 16-hex token
const { data: clientMatch } = await supabase
  .from("clients")
  .select("id, token")
  .eq("reply_token", extractedToken)
  .maybeSingle();

if (clientMatch) {
  // Find most recent non-archived conversation for this client
  const { data: convs } = await supabase
    .from("conversations")
    .select("id, category")
    .eq("client_id", clientMatch.id)
    .neq("category", "archived")
    .order("last_message_at", { ascending: false })
    .limit(1);
  
  let conversationId;
  if (convs && convs.length > 0) {
    // Route to most recent
    conversationId = convs[0].id;
  } else {
    // No active conversation — create new one
    const { data: newConv } = await supabase
      .from("conversations")
      .insert({
        client_id: clientMatch.id,
        category: "conversations",  // lands in active inbox
      })
      .select()
      .single();
    conversationId = newConv.id;
  }
  
  // Insert message into conversation, return success
  // (existing message insert logic)
  return; // Skip the HMAC scan
}

// FALL THROUGH to existing HMAC scan (legacy per-conversation tokens)
```

**Order of operations:**
1. Check Auto-Submitted header → skip if auto-reply
2. Extract token from address
3. Look up `clients.reply_token` (indexed, O(1))
4. If found → route to client's most recent active conversation, or create new
5. If not found → fall through to existing HMAC scan (per-conversation legacy)
6. If still not found → unmatched_inbound

### Phase 3: CRM UI display (~30 min)

In `src/components/crm/ConversationPanel.tsx`:

Add a new line under the existing customer name/email header:

```tsx
<div className="text-sm text-muted-foreground flex items-center gap-2">
  <span className="font-mono">{client.reply_token}@reply.rolliworks.com</span>
  <button onClick={() => {
    navigator.clipboard.writeText(`${client.reply_token}@reply.rolliworks.com`);
    toast.success("Copied to clipboard");
  }}>
    Copy
  </button>
</div>
```

Style to match existing email/phone line — small, muted, secondary text.

### Phase 4: QA (~30 min)

After deploy, test:
1. Open any client conversation → verify reply_token address shows under name/email
2. Click Copy → toast appears, clipboard contains the address
3. Send a test email TO `<reply_token>@reply.rolliworks.com` from a different inbox
4. Verify it lands in that client's RC conversation
5. Send a test email WITH `Auto-Submitted: auto-replied` header → verify it's skipped (no routing)
6. Send a test email TO an OLD per-conversation HMAC token → verify it STILL routes (legacy path intact)

## Acceptance

1. All 259 existing clients have a `reply_token` (run COUNT and verify)
2. New client creation auto-generates `reply_token`
3. Inbound handler routes per-customer tokens correctly
4. Existing per-conversation HMAC tokens still work (no regression)
5. Auto-reply emails (Auto-Submitted header) are skipped
6. CRM displays reply address under client name with Copy button
7. Customer with no active conversation → new conversation created on inbound
8. No errors in edge function logs

## Files touched

- `supabase/migrations/<timestamp>_add_reply_token.sql` (new migration)
- `supabase/functions/portal-intake-router/index.ts` (generate token on client creation)
- `supabase/functions/inbound-email-handler/index.ts` (auto-reply filter + lookup logic)
- `src/components/crm/ConversationPanel.tsx` (UI display)

## Out of scope

- Token rotation UI (manual rotation can be done via DB if needed for now)
- Showing the token to the CLIENT (staff-only display)
- Per-conversation token deprecation (legacy path stays)
- Bounce handling beyond auto-reply filter

## Note on Test branch

Test branch is currently sitting with a known inbox display bug (case-sensitive `team_users` lookup). Building this feature on Test means we'll have two pending fixes before merging to Live. Surface that explicitly when you're ready to merge.

---

End of prompt.
