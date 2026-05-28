# Build — Auto-Email Customer on Inspection Approval Submit

**→ Paste into: RW chat**

---

## Surgical fix only

Touch ONLY `submit-inspection-approval/index.ts`. No other changes.

## Goal

After a customer submits their inspection approval, automatically email them a copy of the signed inspection with a link back to their client portal.

This is in addition to the existing email to `help@rolliworks.com`. Do not change or remove the existing internal-team email.

## Where to add the new email

Inside `submit-inspection-approval/index.ts`, after the existing `inspection_approvals` UPDATE and the existing internal-team email send. Order:

1. Existing: flip status `pending → approved`/`declined` ✓
2. Existing: capture TC audit fields ✓
3. Existing: send internal email to `help@rolliworks.com` ✓
4. Existing: push to RS endpoints ✓
5. **NEW: send customer copy email** ← add here

## Customer email content

**Recipient:** `inspection_approvals.client_email`

**From:** Resend default sender (whatever currently sends the internal email)

**Subject:** `Your Rolliworks Inspection — Approved May 17, 2026` (use actual date from `approved_at`)

**Body (HTML):**

```html
<div style="font-family: Georgia, serif; max-width: 600px; margin: 0 auto; padding: 24px; background: #faf7f2; color: #333;">
  
  <h1 style="color: #047857;">Thank You, [approved_by_name]</h1>
  
  <p>Your inspection for <strong>[watch_brand] [watch_model]</strong> has been received. Here's a summary of what you approved today:</p>
  
  <div style="background: white; padding: 20px; margin: 20px 0; border-radius: 4px;">
    <p><strong>Estimate Number:</strong> [estimate_number]</p>
    <p><strong>Date Approved:</strong> [approved_at formatted]</p>
    <p><strong>Total Approved:</strong> $[total_approved]</p>
    <p><strong>Items Approved:</strong> [items_approved_count] of [total items]</p>
  </div>
  
  <p style="text-align: center; margin: 30px 0;">
    <a href="https://my.rolliworks.com/client/[CLIENT_TOKEN]/c/[CONVERSATION_ID]" 
       style="background: #047857; color: white; padding: 14px 32px; text-decoration: none; border-radius: 4px; font-weight: bold;">
      View Full Inspection Report
    </a>
  </p>
  
  <p style="color: #666; font-size: 14px; margin-top: 40px;">
    You can revisit your inspection report anytime via the link above. We'll be in touch with updates as we work on your watch.
  </p>
  
  <p style="color: #666; font-size: 14px;">
    Questions? Reply to this email or visit your portal.
  </p>
  
  <hr style="margin: 30px 0; border: 0; border-top: 1px solid #ddd;">
  
  <p style="color: #999; font-size: 12px; text-align: center;">
    Rolliworks Inc.<br>
    <a href="https://rolliworks.com" style="color: #999;">rolliworks.com</a>
  </p>
</div>
```

Substitute the bracketed placeholders with real values from the approval / inspection data.

## How to get the RC client token + conversation_id

This is the tricky part. RW doesn't directly know the customer's `rc_client_token` or `conversation_id`.

**Look up via existing RS push pattern:**
- RW already calls RS's `rw-tc-acceptance` and `rw-parts-approved` after approval
- RS knows the customer's `rc_conversation_id` (added today on `customers` table) AND knows the matching RC client
- Best path: extend the RS push to RETURN the rc_client_token + rc_conversation_id back to RW

**Alternative (simpler):** Make a direct call from RW to RC's existing endpoint to look up by email. Or check if `rc_client_token` is already available on the inspection's customer record.

Investigate the cleanest path and propose. If neither path is simple, you can:
- Send the customer email WITHOUT the link (just summary content)
- Flag it as "v2 needs link enrichment"

## Failure handling

If the customer email fails to send (Resend error, bad email, etc.):
- Log the failure but DON'T fail the whole approval
- Approval was already successful, customer email is a nice-to-have

## Acceptance

1. Customer approves an inspection
2. Existing internal email still sends to help@rolliworks.com
3. NEW: Customer receives a copy at their email address with:
   - Subject mentioning their inspection
   - Summary (watch, estimate#, total, item counts)
   - Link to portal (if we figured out token lookup)
4. Approval still completes successfully even if email send fails

## Out of scope

- Don't store the email content (it's a notification, not the artifact)
- Don't change anything about the approval logic itself
- Don't modify RS push behavior
- No PDF attachment (link to portal is the artifact)

## Test

Submit an approval for a test customer. Verify:
- Customer email arrives at their inbox
- Link in email opens their portal correctly
- Internal email still arrives at help@rolliworks.com
- No errors in function logs

## Report

Tell me:
- Where you found the rc_client_token (if you did)
- Whether the email send is implemented with link or without
- Any concerns about the email rendering across clients

---

End of prompt.
