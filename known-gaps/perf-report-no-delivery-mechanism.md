---
silo: 7
date: 2026-05-14
status: known feature gap, not addressed
last_validated: 2026-05-14
---

# Performance Report — no delivery mechanism

## The gap

The Performance Verification Report is clearly designed to be customer-facing:
- Branded sheet styling — gold accent rule, EB Garamond serif, cream "PAPER" background, formal letterhead-like header
- Editorial copy directed at the client ("presented as a concise client summary", "how it is expected to behave in actual daily wear")
- "How to read this" and "The Rolliworks standard" two-column boilerplate
- Footer "BENCH VERIFIED · ROLLIWORKS"
- "Service Narrative" section explicitly "Written for the client" with AI rewrite button "Polish for client"
- Filename embeds client last name (`Performance-Report-{estimate#}-{lastname}-{date}`)

But delivery is entirely manual:
- No email/send action on the preview, entry page, or SO detail
- No `client_photo_request`-style token flow
- No public route, no Supabase storage upload
- Output is local JPG/PDF download or browser print
- Staff must manually attach to an outbound email

## Why it matters

Staff is doing manual labor (download → compose email → attach → send) for every Performance Report that goes to a customer. That's friction every day.

## Possible fixes

1. **"Email to customer" button** on preview that generates PDF, uploads to storage, sends a templated email.
2. **"Send via Customer Portal" / RC integration** — when RC is built (per Section 10 of handoff), reports could appear in the customer's portal automatically.
3. **Both, sequentially.**

## Status

Identified during silo 7 investigation. Real product opportunity, not a fix. Independent of the silo 7 client_first_name addition. Half-day to full-day project when ready.
