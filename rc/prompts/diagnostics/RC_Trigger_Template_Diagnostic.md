# Diagnostic — Trigger Template Not Moving Conversation to Quoted

**→ Paste into: RC chat**

---

## The bug

When admin sends the "Estimate Sent" trigger template (marked `is_trigger=true`, `move_to_category='quoted'`), the conversation does NOT move to the Quoted tab.

The expected flow:
1. Admin in CRM opens a conversation
2. Clicks "Use this" on the "Estimate Sent" template (has a star icon, confirmed marked as trigger)
3. Email sends (or message is composed and sent)
4. **Expected:** conversation.category updates to 'quoted', conversation moves to Quoted tab
5. **Actual:** conversation stays in current category (Conversations, Requests, etc.)

## Investigate the chain

The trigger logic depends on a chain of events. Walk through each step to find the break:

### Step 1: Template detection at click time
- When admin clicks "Use this" on the trigger template, does the system capture which template was used?
- Open the SavedRepliesCarousel component
- Verify: does it pass template_id to the composer when user clicks "Use this"?

### Step 2: template_id on the sent message
- When the message gets sent (Resend out + DB insert into `messages`), is `template_id` actually written to the messages row?
- Query a recent message that should have triggered:
  ```sql
  SELECT id, template_id, author_type, created_at, content 
  FROM messages 
  WHERE conversation_id = '<test conversation>' 
  ORDER BY created_at DESC 
  LIMIT 5;
  ```
- Is template_id NULL or populated?

### Step 3: DB trigger function `route_conversation_on_message`
- Look at the function definition
- Verify the logic: when author_type='team' AND template_id is set, it looks up saved_replies and applies move_to_category
- Test it manually:
  ```sql
  -- Find a known trigger template
  SELECT id, label, is_trigger, move_to_category 
  FROM saved_replies 
  WHERE is_trigger = true;
  
  -- Check if the trigger function fires
  -- (Insert a test message with that template_id and watch the conversation row)
  ```

### Step 4: RLS / permissions
- Can the DB trigger function actually UPDATE conversations.category?
- The function runs with which permissions (definer? invoker?)
- If invoker, does the admin user have UPDATE access on conversations.category?

### Step 5: The conversation refetch
- Even if the category update happened in DB, does the inbox refetch trigger?
- Move to Quoted tab manually — does the conversation appear there now?
- If yes → backend works, frontend not refreshing
- If no → backend issue

## Report back

For each step, tell me:

**Step 1:** Does template_id pass from carousel → composer? (Check the code, not behavior)

**Step 2:** Does template_id get written to the messages row? (Check a real recent message)

**Step 3:** Does the DB trigger function exist and have correct logic? (Show the function definition)

**Step 4:** Does the function have permission to update conversations.category?

**Step 5:** After confirmed sent, does the conversation actually move to Quoted in the DB (even if UI doesn't show it)?

Most likely culprit (in my experience):
- A) template_id never makes it from carousel to messages row (Step 1 or 2)
- B) DB trigger function isn't actually checking saved_replies properly
- C) UI cache not invalidating after the category change

Once we know which step fails, the fix is targeted.

## Test conversation

Use a conversation you can safely move around. Pick one in Conversations or Requests. Send the trigger template. Then run the diagnostics.

---

End of prompt.
