# Diagnostic — "Move to Quoted" Button Not Working

**→ Paste into: RC chat**

---

## Bug

Right-click → "Move to Quoted" was added to the context menu. But clicking it does NOT actually move the conversation to the Quoted tab. The conversation stays in its current category.

## Investigate

1. **Check the click handler** — does the onClick actually run? (browser console log it to verify)

2. **Check the database update** — is the `UPDATE conversations SET category = 'quoted' WHERE id = ?` actually firing? Check:
   - Is there an `await` missing on the supabase call?
   - Is the conversation_id being passed correctly?
   - Any console error?

3. **Check RLS** — does the current user have UPDATE permission on `conversations.category`?
   - Compare to: when staff archives a conversation, that update works. What's different here?
   - Maybe the RLS policy specifically allows `state = archived` but not category changes?

4. **Check the refetch** — after the update, does the inbox list refetch?
   - If update succeeds but UI doesn't refresh, conversation appears stuck

5. **Check the DB trigger** — we added `route_conversation_on_message` to auto-route based on messages. Could that trigger be IMMEDIATELY routing the conversation back to its previous category? (Unlikely since no new message is inserted, but worth verifying it doesn't fire on plain category updates.)

## Report

Tell me:
1. Does the onClick fire? (Add console.log to verify)
2. Does the UPDATE statement run? (Check network/Supabase logs)
3. Any errors thrown?
4. Does the UI refresh after the action?

Once we know which step fails, the fix is small.

---

End of prompt.
