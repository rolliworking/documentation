# Research — Inbox Restructure with Trigger Templates

**→ Paste into: RC chat**

---

# Research mode — no code changes, investigate only

## Context

Big restructure of CRM inbox organization. Need to validate the architecture before building tonight.

## The full state machine

```
[REQUESTS]
    │
    │  Admin sends trigger template (e.g., "Estimate Sent")
    ▼
[QUOTED]  ◄─── from anywhere when trigger sent
    │
    │  Client replies (ANY reply)
    ▼
[CONVERSATIONS]
    │
    │  Staff manually archives
    ▼
[ARCHIVED]

Universal override: ANY client reply, from ANY category, → CONVERSATIONS
```

## Routing rules in plain English

1. New intake_lead from RS/Wix → lands in Requests
2. Stays in Requests until admin sends a trigger template
3. Trigger template sent → moves to Quoted
4. Sits in Quoted awaiting client reply
5. If client never replies → stays in Quoted until manually archived
6. Client replies (from ANY folder) → moves to Conversations
7. Manual archive button → moves to Archived
8. Reply to archived → bumps back to Conversations

## Trigger templates — architectural design

All saved reply templates live in ONE place — same UI as today (the saved replies carousel).

**Editing a template** opens a modal/dialog with a new section:
- Checkbox: "Make this a trigger template" (admin-only — hidden for responders)
- If checked: dropdown for `move_to_category` (Quoted, Conversations, Archived)

**Visibility rules:**
- Personal saved replies (non-trigger): visible only to the creator (current behavior, unchanged)
- Trigger templates: AUTOMATICALLY shared with all staff (brand consistency, workflow automation)

So: marking a template as a trigger auto-promotes it from personal → team-shared.

**Visual indicator:**
- Trigger templates display with a star icon next to the template name in the saved replies carousel
- Tooltip on hover: "Sending this template moves the conversation to [category]"

**When staff sends a trigger template:**
1. Email gets sent normally
2. ALSO: conversation.category updates to `move_to_category`

## Labels (replaces current flag color system)

Multi-label support. Each conversation can have any combination of:
- yellow = "update work order"
- red = "Vienna"
- green = "Mike"

Displayed as small colored dots inline with conversation name.

Drop existing flag color data entirely (start fresh).

## Unread count

Numbered circle showing unread CLIENT messages count.
- 1 = one unread client message
- 2 = two unread
- 0 = no badge

Count = messages where `from = client AND created_at > user's last_read_at for that conversation`.

## Investigate

### 1. Current inbox structure
- Where is the inbox list rendered? (`CrmInbox.tsx`?)
- What columns exist on `conversations` today?
- How does the current Active/Flagged/Yellow/Archived filter work?
- Are flags stored as `flag_color` column on conversations? Or separate table?

### 2. Saved replies architecture
- Confirm: `saved_replies` table is per-user via `user_id` column
- Plan: add columns `is_trigger boolean`, `move_to_category text`, `is_shared boolean` (auto-true when is_trigger=true)
- Update RLS: trigger/shared templates visible to ALL authenticated, personal ones still creator-only
- Admin role required to set is_trigger=true

### 3. How template usage gets detected
- When staff clicks "Use this" on a saved reply, what currently happens?
- Does the rendered text get sent, or does the system know "this came from template X"?
- For trigger logic: we need to know WHICH template was sent
- Recommend: add `template_id` column to `messages` to record source template

### 4. Hook points for category routing
- When `portal-intake-router` creates a conversation → set category = 'requests'
- When `inbound-email-handler` adds a message from client → set category = 'conversations'
- When staff sends a trigger template → set category from template metadata

Walk through where each of these hooks would live.

### 5. Unread count complexity
- Current `conversation_read_states` only tracks boolean `is_read`
- Need: count of CLIENT messages since user's last_read_at
- Could add `last_read_at timestamptz` to `conversation_read_states`
- Performance: query runs per conversation in inbox list (potentially 100+ rows) — is this OK or do we need a counter cache?

### 6. UI replacements
- Replace tabs: Unread/Active/Flagged/Yellow/Archived → Conversations/Requests/Quoted/Archived
- Replace flag color buttons in conversation view → label picker (color dots)
- Replace flag color dots in list → multi-label color dots
- Replace unread visual (bold + bg color) → numbered circle badge
- Add star icon to trigger templates in saved replies carousel

## Specific concerns to address

1. **Template detection reliability**: if we mark a template as trigger, does the click-to-use mechanism reliably record the template_id on the sent message? Or does staff sometimes manually paste content that bypasses the trigger?

2. **Manual override**: should there be a "Move to Quoted" button for cases where staff didn't use the trigger template but still wants to move the conversation? (Could be deferred to v2)

3. **Auto-share enforcement**: when is_trigger flips to true, should the system AUTOMATICALLY set is_shared=true? Or warn admin and require them to also tick is_shared?

4. **What happens to existing flag data**: drop entirely per user direction. Confirm no surprises in code that references it.

## Build phases (target: ~8 hours)

- Phase 1: Schema (categories on conversations, multi-label table, trigger fields on saved_replies, template_id on messages) (~1.5 hrs)
- Phase 2: Trigger template logic + UI (admin checkbox in edit, star icon in carousel) (~2 hrs)
- Phase 3: Category routing hooks (intake_router, inbound_handler, trigger send) (~1.5 hrs)
- Phase 4: Labels system + UI (color dots) (~1.5 hrs)
- Phase 5: Unread count + UI (numbered badge) (~1 hr)
- Phase 6: Inbox tabs restructure (~30 min)
- Phase 7: Testing (~30 min)

## Report back

- Current state of each subsystem
- Architectural concerns or simpler approaches
- Realistic time estimate per phase
- Anything that pushes us OVER 8 hours — flag for scope cuts

Don't apply changes. Research only.

---

End of prompt.
