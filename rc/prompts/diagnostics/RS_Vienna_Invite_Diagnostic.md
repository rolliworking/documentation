# Research — Vienna's Send Portal Invite Error (RS-side investigation)

**→ Paste into: RS chat**

---

# Research mode — no code changes, investigate only

## Context

Vienna (help@rolliworks.com) gets a 22xx error when clicking Send Portal Invite from RS customer detail page. Happens for ALL customers she tries.

Mike does NOT hit this error.

RC investigated and found two edge functions on RC side that reference a dropped column from this morning's migration. RC patched those edge functions. **But Vienna is STILL getting the error.**

That means the failure is happening BEFORE the call reaches RC's edge function. Likely RS-side.

## What you've established before

- Vienna's user record: 8a92dfd0-… with role=responder (per earlier RC research)
- Mike's user record: 976c0e7f-… with role=admin
- help@rolliworks.com user: 5dea… with role=admin

WAIT — there might be confusion here. Is Vienna's user the `8a92dfd0` (responder) OR the `5dea` (help@ admin)?

If Vienna logs in as help@rolliworks.com, she's the admin. If she logs in with her personal email, she's the responder.

## Investigate

### 1. Identify which user record Vienna actually uses

- Which email is Vienna logging in with when she hits this error?
- What's her ACTUAL user_id in RS auth.users?
- What's her role in team_users?

### 2. Walk through the Send Portal Invite flow on RS side

Find the button/handler that triggers Send Portal Invite. Trace what RS does:
- Does RS check user permissions before calling RC?
- Are there any RLS policies that gate access to customer data needed for the invite?
- What data does RS fetch from its own DB before calling RC's endpoint?
- What auth context is passed to RC?

### 3. Specific RS-side failure points to check

a) **RLS on customer data**: when Vienna's session reads customer data to send to RC, does her role have SELECT permission on:
   - `customers` table
   - `estimates` table
   - Any other intake_lead data being replayed?

b) **Service key vs user JWT**: does RS call RC using:
   - Vienna's user JWT (which would require her to have permission)
   - Or a service role key (which would bypass user perms)
   
   If user JWT, what fields/columns does Vienna need access to that she might lack?

c) **Customer detail page**: does Vienna even have permission to OPEN the customer detail page? If yes, the button click — does RS run a different permission check than the page-open check?

### 4. RS error logs

Check the RS edge function logs (if Send Portal Invite calls an edge function on RS first):
- Find Vienna's most recent failed attempt
- Get the EXACT error code (might be 42501 for permission denied, 22xxx for data issue, or something else)
- Get the full error message text
- Identify which RS query/table triggered it

### 5. Compare Vienna and Mike's experience

- If Mike clicks Send Portal Invite on the SAME customer, does it work?
- If yes → 100% Vienna-specific, role/permission related
- If no → the bug is broader

## Specific output wanted

1. Exact error code (e.g., 42501 vs 42703 vs 22001 vs 22023)
2. Where the failure happens (RS edge function? RS RLS? After call to RC?)
3. Vienna's actual role per her user_id
4. Whether Vienna can OPEN the customer detail page at all (or only click and fail)
5. If permission-related: which specific table/column is denied

## Don't apply fixes yet

Just diagnose. Return findings.

---

End of prompt.
