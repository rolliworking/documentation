# Diagnostic — Vienna Can't See Messages

**→ Paste into: RC chat**

---

## Investigation only — no code changes yet

## The problem

Vienna (help@rolliworks.com) successfully logged into the CRM at https://my.rolliworks.com/crm/inbox. She's an authenticated user. But her inbox shows NO messages.

We expected RLS to grant all authenticated team users full read access.

## Investigate

1. **Verify Vienna's team_users row exists**
   - Query `team_users` where email = 'help@rolliworks.com'
   - Confirm: does the row exist?
   - What `user_id` does it have? Does it match her auth.users record?
   - What role is set?

2. **Verify Vienna's auth.users record**
   - Query auth.users where email = 'help@rolliworks.com'
   - Get her user_id
   - Compare to team_users.user_id — do they match?

3. **Check RLS policies on conversations and messages**
   - List current RLS policies on `conversations` table
   - List current RLS policies on `messages` table
   - Specifically: what's the SELECT policy condition?

4. **Test the policy logically**
   - Run a query AS Vienna's user_id: `SELECT * FROM conversations LIMIT 5`
   - Does it return rows?
   - If 0 rows, what policy is filtering them out?

## Common causes to check

- team_users row created but user_id is NULL or wrong (foreign key not linked)
- RLS policy requires a row in team_users matching auth.uid() — if Vienna's row was inserted without the right user_id linkage, the policy filters her out
- Email mismatch (case sensitivity, whitespace)
- Wrong entity_access value

## Report back

Tell me:
1. Vienna's team_users row contents (paste row)
2. Vienna's auth.users.id (paste UUID)
3. The current SELECT policy on `conversations` (paste SQL)
4. Whether the user_id values match between team_users and auth.users

Then we'll know exactly what to fix.

---

End of prompt.
