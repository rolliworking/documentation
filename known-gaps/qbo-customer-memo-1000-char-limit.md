---
silo: 8
date: 2026-05-15
status: open — blocking SO 501718 push
last_validated: 2026-05-15
---

# QBO Customer Memo field exceeds 1,000 char limit on SO push

## What

Push from RS Sales Order Fulfill page to QBO returns ValidationFault on SO 501718:

```
QuickBooks API error: {"Fault":{"Error":[{
  "Message":"String length is either shorter or longer than supported by specification",
  "Detail":"String length specified does not match the supported length. Min:0 Max:1,000 supported. Supplied length:1,016",
  "code":"2050",
  "element":"Customer Memo"
}],"type":"ValidationFault"}}
```

16 characters over the 1,000-char cap. QBO enforces this server-side; truncation must happen RS-side before push.

## Affected SO

- SO 501718 — Jack Matiasevich, 15 lines, $2675. Blocked silo 8.

## What we don't know

- Which field(s) in RS get concatenated into the Customer Memo on QBO push. Candidates: customer notes, SO line item descriptions concatenated together, auto-generated memo, sales order memo.
- Whether truncation logic exists anywhere in `qbo-sales-order-push` (frozen file per STABILITY_RULES.md from earlier sessions).
- Whether other SOs are at risk — anything with a heavy memo or many long line descriptions could trip the same limit.

## How to investigate

Read `supabase/functions/qbo-sales-order-push/index.ts` (frozen file — read-only OK) for the Customer Memo construction. Identify the source field(s). Then either:
- Truncate at push time (RS-side fix, may need to lift frozen-file restriction explicitly for this surgical patch)
- Truncate at memo-entry time (catches it earlier, applies to all future pushes)

## Why deferred

Surfaced silo 8 at end of session; not addressed. Customer-facing blocker — should be top of next session.
