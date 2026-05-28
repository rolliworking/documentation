# Roadmap

**Last updated:** May 27, 2026

Engineering priorities, organized by horizon.

---

## Immediate cleanup (next session)

1. **Remove abandoned internal notes** — disable `detectInternalMention` (return false) or revert; keep Mark Unread bump. CRITICAL: stops customer-facing leak.
2. **Restore reply token display** in CRM header (lost in merge)
3. **Verification pass** — confirm today's shipped items actually work:
   - Shared team read state (Mike reading clears Vienna's badge)
   - Inbox case-insensitive lookup (both see populated inbox)
   - RS role allowlist (Vienna can Send Portal Invite)
   - Row color coding restored

---

## This week

- **Customer ID architecture decision** — dedicated session; stabilize RS↔RW↔RC linkage
- **RS→RW data integrity (Task 2)** — separate chat; fix the urgent missing field, then service_type/date/auth/customer-ID
- **Band flow split (Task 1)** — RW-owned; customer sees correct flow (band omits In Testing). In progress, commit `98ce8a3`
- **Estimate # lookup** — smart search bar (Option 1); decide storage approach (RC→RS call vs cache)
- **Booking link from RC** — RS research pending; reuse existing RS Send Booking Link via Resend with reply token
- **Reply token end-to-end verification** — confirm Postmark delivery + CC routing
- **Welcome email layout verification**
- **Trigger template fix** (depends on diagnostic)

---

## Memorial Day weekend (large items, ~14-20 hrs)

| Item | Effort |
|------|--------|
| Email forwarding to RC + existing customer detection | 3-5 hrs |
| Inbound Inquiries view + Convert to Service Request | 2-3 hrs |
| Customer intelligence layer (LTV + notes + ratings) | 4-5 hrs |
| Status change notifications full system (RW templates → Resend → reply token, opt-out, backward-move handling) | 7-10 hrs |
| Multi-job progress display (accordion, one pattern desktop+mobile) | 3 hrs |
| Keyboard workflow bundle (Delete + Arrows + Label filter; Answered already shipped) | 4-5 hrs |
| Backup hardening (GitHub sync, off-platform exports, tested restore) | 2-3 hrs |

Suggested split:
- Saturday: Email forwarding + existing customer detection
- Sunday: Customer intelligence + backup hardening
- Monday: Multi-job + Inbound Inquiries + buffer

---

## Status notifications — locked architecture

- RW owns email templates (existing mailto infrastructure)
- Replace mailto with Resend auto-send to customer with reply token in Reply-To
- RW triggers webhook to RC on status change (including component lookup)
- RC sends via Resend + adds message to conversation thread
- Customer opt-out mechanism in RC
- Each milestone email sent ONCE (backward moves don't retrigger)
- Process map shows backward movement; no email spam
- Labels: Polishing → Polish/Clean; Shipped → Shipping/Pick Up
- NO template editor in RC (reuse RW assets)

---

## Future (captured, not scheduled)

- Slack integration (only if internal team need emerges — currently parked; "internal notes in RC" was the alternative, now abandoned)
- AI extraction from emails (Gemini summary at top of inspection)
- Inspection PDF generation / print
- CRM desktop sidebar mirror of portal
- Photos data wiring (needs RS photo endpoint)
- Snapshot inspection to Storage for legal-grade contract integrity
- Per-customer reply token rotation UI
- Quote shopping handling (multiple estimates, one acceptance)
- Manual estimate # entry
- Replace Wix intake (long-term)

---

## Documentation discipline (this repo)

- Maintain Master Knowledgebase + per-repo READMEs
- Update docs alongside code changes
- Write session handoffs when state changes significantly
- Generate READMEs for RS, RW, RC via Cursor (prompt in prompts/)
