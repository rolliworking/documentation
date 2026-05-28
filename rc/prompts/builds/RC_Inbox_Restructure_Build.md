# Build — Inbox Restructure: Categories, Labels, Trigger Templates, Unread Count

**→ Paste into: RC chat**

---

## Surgical fix only

Touch ONLY the files needed for this specific change. Don't refactor unrelated code. Don't add "while we're here" improvements. Minimal blast radius. If you spot other issues, flag them at the end but don't fix them.

## Decisions locked from research

| Decision | Value |
|---|---|
| Categories | Conversations / Requests / Quoted / Archived |
| State machine | New intake → Requests. Trigger sent → Quoted. Any client reply → Conversations. Manual archive → Archived. Reply to archived → Conversations. |
| Trigger templates | Live in saved_replies, admin-only checkbox in edit modal |
| Trigger visibility | Trigger templates auto-shared to all staff (no is_shared field, derive from is_trigger) |
| Trigger firing | Immediately on message insert |
| Template detection | Pass template_id from carousel → composer → message insert |
| Labels | Fixed three colors: yellow=update work order, red=Vienna, green=Mike |
| Label UI | Right-click context menu on conversation (labels + archive) |
| Unread count | Numeric badge showing count of client messages since last_read_at |
| Existing flag data | Drop entirely (start fresh) |

## Phase 1: Schema (~1.5 hr)

### Migrations

```sql
-- Categories on conversations
alter table conversations add column category text default 'conversations'
  check (category in ('conversations','requests','quoted','archived'));

-- Drop the old flag system entirely
alter table conversations drop column flag;
alter table conversations drop column flag_set_by;
alter table conversations drop column flag_set_at;

-- New labels table (multi-label support)
create table conversation_labels (
  conversation_id uuid references conversations(id) on delete cascade,
  label text not null check (label in ('yellow','red','green')),
  set_by uuid references auth.users(id),
  set_at timestamptz default now(),
  primary key (conversation_id, label)
);
alter table conversation_labels enable row level security;
create policy "team can manage labels" on conversation_labels
  for all using (auth.uid() in (select user_id from team_users));

-- Trigger fields on saved_replies
alter table saved_replies add column is_trigger boolean default false;
alter table saved_replies add column move_to_category text 
  check (move_to_category in ('conversations','requests','quoted','archived'));

-- Update saved_replies RLS to include shared trigger templates
drop policy if exists "users see own saved replies" on saved_replies;
create policy "users see own or trigger templates" on saved_replies
  for select using (auth.uid() = user_id OR is_trigger = true);
-- Keep create/update/delete as user_id = auth.uid() for personal,
-- admins manage triggers via their own user_id but mark them is_trigger=true

-- Only admins can set is_trigger=true (enforce in app code, not RLS, for simplicity)

-- template_id on messages for trigger detection
alter table messages add column template_id uuid references saved_replies(id);

-- last_read_at already exists on conversation_read_states per research
```

### Backfill

Set all existing conversations to `category = 'conversations'` (default does this).

## Phase 2: Trigger template UI (~1.5 hr)

In the saved replies edit modal (`SavedReplyModal.tsx`):
- Add section visible ONLY if current user role is 'admin':
  - Checkbox: "Make this a trigger template"
  - If checked, show dropdown: "Move conversation to: [Conversations / Requests / Quoted / Archived]"
- Save: writes `is_trigger` + `move_to_category` to row

In `SavedRepliesCarousel.tsx`:
- If `is_trigger === true`, render a star icon next to template name
- Tooltip on hover: "Sending this template moves the conversation to {move_to_category}"

## Phase 3: Category routing hooks (~1.5 hr)

### A) Intake creates Requests

In `supabase/functions/portal-intake-router/index.ts`:
- When creating new conversation, set `category: 'requests'`

### B) Client reply moves to Conversations

DB trigger on `messages` insert (cleanest per research):
- If `author_type = 'client'` → set parent `conversations.category = 'conversations'`
- Also bumps from Archived back to Conversations (per spec)

### C) Trigger template send moves to category

Same DB trigger, additional logic:
- If `author_type = 'team'` AND `messages.template_id` is set:
  - Look up `saved_replies.is_trigger` and `saved_replies.move_to_category` for that template_id
  - If `is_trigger = true`, set conversation category to `move_to_category`

### D) Manual archive

Existing archive button stays. Just update it to write `category = 'archived'` instead of `state = 'archived'`. Or keep `state` AND `category` in sync — but `category` is the new source of truth.

## Phase 4: Labels system + right-click UI (~2 hr)

In `InboxList.tsx`:
- For each conversation row, query labels from `conversation_labels`
- Render small colored dots inline with conversation name (multiple dots if multiple labels)
- Use right-click handler (`onContextMenu`) on each conversation row

Right-click context menu pops up with:
```
● Yellow (update work order)
● Red (Vienna)
● Green (Mike)
──────────
Archive
```

Clicking a color:
- If already labeled with that color → remove it (delete from conversation_labels)
- If not labeled → add it (insert into conversation_labels)

Clicking Archive → existing archive logic (writes category = 'archived')

Multi-label means multiple dots can show on one conversation.

## Phase 5: Unread count + UI (~1 hr)

### Backend RPC or view

Create SQL function or view that returns per-conversation unread count for the current user:

```sql
create or replace function get_inbox_unread_counts(team_user_id uuid)
returns table(conversation_id uuid, unread_count bigint) as $$
  select m.conversation_id, count(*) as unread_count
  from messages m
  left join conversation_read_states crs 
    on crs.conversation_id = m.conversation_id 
    and crs.user_id = team_user_id
  where m.author_type = 'client'
    and m.created_at > coalesce(crs.last_read_at, '1970-01-01'::timestamptz)
  group by m.conversation_id;
$$ language sql stable;
```

### Frontend

`useInboxRows` fetches unread counts via this RPC and merges into row data.

`InboxList.tsx` renders:
- If unread_count > 0 → small circle badge with the number next to conversation
- If 0 → no badge
- Replace existing "bold + bg color" unread visual

## Phase 6: Inbox tabs restructure (~30 min)

In `CrmInbox.tsx`:
- Replace tabs: Unread/Active/Flagged/Yellow/Archived → Conversations/Requests/Quoted/Archived
- Filter inbox rows by `conversations.category` (not derived state)
- Default tab: Conversations
- Drop yellow flag tab entirely

## Phase 7: Testing (~30 min)

End-to-end test:
1. New intake (test customer) → lands in Requests ✓
2. Admin sends trigger template → moves to Quoted ✓
3. Test customer replies → moves to Conversations ✓
4. Right-click conversation → label as yellow → dot appears ✓
5. Right-click conversation → label as red → both dots appear ✓
6. Right-click → Archive → moves to Archived ✓
7. Test customer replies again → back to Conversations ✓
8. Unread count: send 2 client messages → badge shows "2" ✓
9. Open conversation → badge disappears ✓

## Out of scope (defer)

- Manual "Move to Quoted" button (no trigger template used) — v2
- User-extensible labels (custom colors/names) — v2
- Label filters in inbox (show only yellow-labeled conversations) — v2
- Notifications when conversation moves between categories — v2
- Mobile-friendly label UI (right-click doesn't work) — v2

## Acceptance criteria

1. Four tabs visible: Conversations / Requests / Quoted / Archived
2. New intake lands in Requests
3. Trigger template send moves conversation to Quoted
4. Client reply bumps to Conversations (from any category, including Archived)
5. Right-click conversation opens context menu with label colors + Archive
6. Multi-label support visible as multiple dots
7. Numeric unread count badges on conversations with unread client messages
8. Old flag data dropped, no UI references to it
9. Star icon visible on saved replies marked as trigger templates
10. Admin-only checkbox for marking trigger templates

## Tell me when ready

After deploying:
- Verify all 9 acceptance criteria above
- Tell me about any issues that came up
- Flag any tech debt or "while we're here" improvements you noticed but didn't apply

---

End of prompt.
