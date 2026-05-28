# Diagnostic — Reply Token CC Routing Not Working

**→ Paste into: RC chat**

---

## Bug

Test sent FROM mike@rolliworks.com TO Gmail (external address) WITH CC to 27864080d72c3bec@reply.rolliworks.com.

Use case: staff replying to an existing customer email thread, CC'ing the reply token to bring the thread INTO RC.

**Result: email did NOT land in the customer's RC conversation.**

## Check inbound-email-handler logs

For the last 5 minutes:

1. Did Postmark deliver this CC'd email to the inbound webhook? (Look for POST to the function)
2. If yes: what's in the request body? Specifically the From, To, CC fields
3. Was the token "27864080d72c3bec" extracted correctly from the CC address?
4. Did the per-customer token lookup find a client?
5. What was the routing outcome? (added to conversation? unmatched_inbound? filtered? rejected?)
6. If rejected/skipped: what was the reason?

## Possible root causes I'm guessing at

1. **Sender mismatch check** — mike@rolliworks.com isn't the customer's email, so check fails
2. **Postmark only parses TO field for tokens, not CC** — token in CC isn't extracted
3. **Postmark Inbound webhook URL misconfigured** — emails never reach RC
4. **Auto-Submitted filter triggering incorrectly** — Outlook headers tripping it
5. **Token DB mismatch** — value in DB doesn't match what's displayed in UI

## Critical design question

The use case is: staff brings external email threads INTO RC by CC'ing the reply token. Sender will OFTEN be `mike@rolliworks.com`, `help@rolliworks.com`, or some other staff/customer email — NOT necessarily the customer's email.

The feature should:
- Accept inbound from ANY sender (not just the matched customer)
- Mark the message attribution correctly:
  - If FROM matches customer email → from client
  - If FROM is staff (rolliworks.com domain) → from team
  - Otherwise → from external (third party, copied on the thread)
- Land in the customer's RC conversation regardless of sender

If sender-mismatch is currently DROPPING these emails, that's a bug — needs to be a flag, not a reject.

## Report

Don't fix yet. Report findings:
- Did email reach inbound-email-handler? (Yes/No)
- If yes: what routing decision was made and why?
- Current sender-handling logic — does it drop on mismatch or just flag?
- What needs to change to support the staff-CC use case?

---

End of prompt.
