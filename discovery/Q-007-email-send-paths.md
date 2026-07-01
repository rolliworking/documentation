# Q-007 — Inventory email send paths + template usage

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-29
**Depends on:** Q-001
**Human answers applied:** D-020 (silent failure banned)
**Inputs read:**
- `apps/rs/supabase/functions/send-*/` (19 functions)
- `apps/rs/supabase/functions/qbo-payment-webhook`, `qbo-invoice-sync`
- `apps/rs/src/pages/setup/EmailTemplatesPage.tsx`
- `apps/rs/src/pages/intake/IntakeEmailPage.tsx`, `sales/PickupStationPage.tsx`
- `documentation/workflows/VIANNA-WORKFLOW.md`, `MICHAEL-WORKFLOW.md`
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**RS outbound email layer** — Every customer-facing and staff-notification email RolliSuite sends via Resend (`quotes.rolliworks.com`), which `message_templates` row each uses, what triggers it, and whether success/failure is observable (D-020).

---

## 2. Email inventory (19 `send-*` functions)

| Function | Template (DB name) | From address | Trigger |
|----------|-------------------|--------------|---------|
| `send-estimate-email` | *(inline/hardcoded)* | `noreply@quotes.rolliworks.com` | UI: estimates, drop-off receipt, intake email |
| `send-shipping-label-email` | `Shipping Label` (UI-loaded) | noreply | Ship station, label requests |
| `send-shipping-info-request` | `Shipping Info Request` | noreply | SO detail, QBO webhook, invoice-sync failsafe |
| `send-fedex-pickup-notice` | `FedEx Pickup Location Notice` | noreply | FedEx dialog |
| `send-pickup-reminder` | `Pickup Reminder` | noreply | SO detail (manual) |
| `send-pickup-reminder-sms` | `Pickup Reminder SMS` | Twilio | SO detail |
| `send-payment-reminder` | `Payment Reminder` | noreply | SO detail |
| `send-payment-reminder-sms` | `Payment Reminder SMS` | Twilio | SO detail |
| `send-pickup-pass` | `Pickup Pass` | noreply | Pickup QR dialog |
| `send-pickup-email` | — | — | **MISSING** — invoked by PickupStationPage |
| `send-lead-followup` | `Intake Lead Follow-Up` | noreply | Intake leads page |
| `send-waitlist-email` | *(hardcoded)* | noreply | Waitlist |
| `send-booking-link` | `Appointment Booking Link` | noreply | Customer detail |
| `send-bypass-optin` | `Shipping Bypass Opt-In` | noreply | Customer detail |
| `send-photo-request` | `Photo Request` | noreply | Customer detail |
| `send-appraisal-email` | `Appraisal Email` *(migration: Appraisal Report)* | noreply | Appraisals |
| `send-po-email` | `PO Email` (UI) | `purchasing@` noreply | PO dialog |
| `send-receive-report` | *(UI body)* | purchasing noreply | PO receive |
| `send-invite` | *(hardcoded)* | noreply | Users admin |

**Ghost:** `send-so-email` in `config.toml` + seeded template `Work Completed - Sales Order` — **no function directory**.

**Non-send-* email paths:** `qbo-payment-webhook`, `qbo-invoice-sync` (auto shipping-info), `return-shipping-info`, `request-shipping-label`, `shipping-bypass-optin` (staff alerts), `rc-portal-invite` (delegates to RC).

---

## 3. Domain and reply behavior

- **Sending domain:** `quotes.rolliworks.com` (all customer email)
- **Reply-to:** `help@rolliworks.com` on most customer sends
- **Pain point (Vianna):** `noreply@` → clients reply anyway → action items lost (no RC threading)
- **BCC:** `help@`, `mike@`, `admin@` on selected templates — not unified CC/BCC into RC (W-44)

---

## 4. D-020 silent-failure audit

| Category | Functions | Risk |
|----------|-----------|------|
| **Observable** | `send-estimate-email`, `send-pickup-reminder`, `send-payment-reminder`, `send-lead-followup`, `send-appraisal-email`, `send-fedex-pickup-notice`, `send-po-email` | Returns error to UI; some write `audit_log` |
| **Fire-and-forget** | `send-bypass-optin`, `send-shipping-info-request`, `qbo-payment-webhook` email helper, `qbo-invoice-sync` failsafe, `shipping-bypass-optin`, `return-shipping-info` | HTTP 200 even if Resend failed |
| **Broken** | `send-pickup-email` | Invoke fails — no function deployed |
| **Missing** | `send-so-email`, bulk testing-complete | Workflow gap |

### Recommended audit layer (rebuild)

1. `email_send_log` table: `id`, `function_name`, `template_name`, `recipient`, `resend_id`, `status`, `error`, `triggered_by`, `entity_type`, `entity_id`, `started_at`, `completed_at`
2. INSERT `status: started` **before** Resend call; UPDATE on response (D-020)
3. No customer-facing path returns 200 without log row in terminal state

---

## 5. Workflow pain point mapping

| Pain point | Source | RS status | Gap |
|------------|--------|-----------|-----|
| noreply replies lost | Vianna §4 | All `send-*` | No RC inbound for transactional sends |
| No CC/BCC into RC | W-44 | — | Architectural |
| Bulk status update emails | Vianna wishlist | `IntakeEmailPage` partial | No unified bulk sender |
| Testing-complete bulk email | Michael §3 | **RW only** | No RS bulk path |
| Invoice + payment link after testing | Michael §3 | Template exists | `send-so-email` missing |
| Pickup code resend | Pickup Station | `send-pickup-email` | **Function missing** |
| Inspection photos RS→RC | Vianna §4 | — | No send function |
| Automated reminders cron | — | Manual only | No scheduled pickup/payment reminders |

---

## 6. Open questions

### Q-007-A: Should customer emails route replies back into RolliConnect?

**Type:** architectural
**Question:** Most outgoing emails send from a "noreply" address, but customers reply anyway — and those replies don't show up in RolliConnect. Should rebuild emails use a reply address that automatically lands in the customer's RolliConnect conversation thread?
**Why it matters:** Vianna's top email pain point: staff miss client replies to shipping notifications and estimates because they go to a general inbox instead of the customer file.
**What I observed:** Every customer email uses noreply@ with reply-to set to help@. RolliConnect has a token-based reply routing system for portal messages but transactional emails don't use it. *(Technical: W-44 CC/BCC into RC threads.)*
**My best guess:** Rebuild puts a unique reply token in the Reply-To header so replies attach to the right RC conversation.
**Default if no answer in 7 days:** Rebuild uses per-send RC conversation token in Reply-To header.

### Q-007-B: Who should send the bulk "testing complete" email — RolliSuite or RolliWorking?

**Type:** scope
**Question:** Michael wants a one-button bulk email when multiple watches finish testing. The email template lives in RolliSuite, but watchmakers work in RolliWorking. Should the button live in RolliWorking (calling RolliSuite to send) or should RolliSuite own the whole flow?
**Why it matters:** Wrong placement means watchmakers can't reach the feature, or duplicate email infrastructure gets built.
**What I observed:** Template "Work Completed - Sales Order" exists in RolliSuite's template table but the send function was never built. Testing-complete workflow today is manual. *(Technical: send-so-email missing; RW status complete trigger.)*
**My best guess:** Button in RolliWorking; RolliWorking calls a RolliSuite send function with an audit log per D-020.
**Default if no answer in 7 days:** RW UI triggers RS send-testing-complete-batch with observable log.

---

## 7. Acceptance criteria check

- ✅ Every `send-*` function cataloged with template and trigger
- ✅ Pain points mapped to Vianna/Michael workflows
- ✅ D-020 fire-and-forget vs observable classified
- ✅ Audit layer recommendation concrete

---

_End of discovery. Q-007 complete._
