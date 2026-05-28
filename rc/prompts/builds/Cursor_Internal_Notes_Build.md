# Cursor Build — Internal Notes + Mark Unread Bump to Top

**→ Paste into: Cursor chat**

---

## Context

You're working in the RC repo (`~/rolliworks/rollicrm`). Two related fixes:

1. **Internal team notes** — staff can write messages to each other inside a customer conversation without the customer seeing them
2. **Mark Unread bumps to top** — when staff marks a conversation unread, it should bump to the top of the Conversations tab list

Both are in production-mid-build. Read carefully — this involves schema, RLS, edge function, AND frontend changes.

## What I want you to do

Read the relevant code FIRST, then plan, then execute. Don't apply changes until you've shown me the plan.

## Feature 1: Internal Notes

### Detection rule
Messages containing `@mike`, `@vienna`, or `@<any-team-user-first-name>` are internal notes.

Use case: "@mike can you help John source new hands" — internal, staff only.

### Schema change
Add column to `messages` table:
```sql
alter table messages add column is_internal boolean not null default false;
create index idx_messages_is_internal on messages(conversation_id, is_internal);
```

### Where the detection happens
In the message send path (likely `ConversationPanel.tsx` send function), BEFORE inserting the message:
- Pattern check: regex like `/@\w+/` for any @ mention
- Cross-reference: check if mentioned name matches any `team_users.first_name` (case-insensitive)
- If match → set `is_internal: true` on insert
- If no match → normal message (could be customer's name like "John")

```ts
async function detectInternalMention(body: string): Promise<boolean> {
  const mentionMatch = body.match(/@(\w+)/);
  if (!mentionMatch) return false;
  
  const mentionedName = mentionMatch[1].toLowerCase();
  const { data: teamMembers } = await supabase
    .from("team_users")
    .select("first_name");
  
  return teamMembers?.some(
    m => m.first_name.toLowerCase() === mentionedName
  ) ?? false;
}
```

### Effects of `is_internal = true`
1. Message inserts to DB with flag
2. Customer portal MUST NOT see this message
3. No outbound notification email to customer
4. No reply-token email
5. Conversation state DOES NOT bump to "awaiting_client"
6. UI in CRM renders this message visually differently (e.g., yellow/amber background, "Internal note" label)

### RLS update
The portal client query for messages needs to filter out is_internal=true:
- Find where the portal reads messages for the client (likely `ClientConversation.tsx` and any anon-accessible message queries)
- Add filter: `.eq("is_internal", false)` or via RLS policy
- Safer: update RLS policy on `messages` table to enforce — anon role cannot SELECT rows where is_internal=true

```sql
create policy "Anon cannot read internal messages"
  on messages for select
  to anon
  using (is_internal = false);
```

Note: this means BOTH the regular anon read policy AND this filter need to work together (intersection).

### Visual treatment in CRM
In `ConversationPanel.tsx` where messages are rendered, add conditional styling:
- Internal note background: amber/yellow tint (e.g., `bg-amber-50` or similar Tailwind)
- Small label: "Internal note" or "🔒 Internal"
- Subtle distinction from regular messages

### Outbound notification skip
In the `send` function, if message is internal:
- Skip the `portal-send-reply-notification` edge function call
- Skip the state update to "awaiting_client"
- Skip the nudge queue insert

## Feature 2: Mark Unread Bumps to Top

### Behavior
When user clicks Mark Unread on a conversation:
1. (Existing) Update `conversation_read_states` to mark unread
2. (Existing) Update `conversations.category = 'conversations'`
3. (NEW) Update `conversations.last_activity_at = now()` so it sorts to top

The Conversations tab presumably sorts by `last_activity_at DESC` (verify in `useInboxRows.ts`). Updating that field bumps the conversation to the top.

### Find and update
- The current Mark Unread handler — should be in `ConversationPanel.tsx` `markUnread` function
- Add `last_activity_at: new Date().toISOString()` to the conversations update

## Acceptance

### Internal Notes
1. Staff writes "@mike can you help" → message saves with is_internal=true → renders with internal styling
2. Customer opens portal → does NOT see the internal note
3. No email goes to customer
4. Regular message without @mention → behaves normally
5. Message says "John needs help" (customer name, not team) → NOT internal
6. Mike sees Vienna's @mike message → he can reply normally

### Mark Unread
1. Conversation in Archived → Mark Unread → moves to Conversations + appears at top of list
2. Conversation already in Conversations → Mark Unread → bumps to top of list
3. Other conversations stay in their previous order

## Workflow

1. Read these files first and report back what you find:
   - `src/components/crm/ConversationPanel.tsx` (send + markUnread functions)
   - `src/hooks/useInboxRows.ts` (how does it sort?)
   - `src/pages/portal/ClientConversation.tsx` (how does portal read messages?)
   - `supabase/migrations/` (look at recent message table migrations)
   - RLS policies on messages table

2. Show me your plan in 4-5 bullet points

3. Wait for my approval

4. Then execute step by step:
   - Migration first
   - Then send logic
   - Then portal RLS/filter
   - Then UI styling
   - Then markUnread bump

5. Commit and push to test branch when complete

## Risks to surface

If you find issues like:
- @mention regex collides with email addresses
- RLS would also block staff from reading internal notes
- Existing message queries don't filter is_internal
- Tests would need updating

Flag them before applying. Don't silently work around problems.

---

End of prompt.
