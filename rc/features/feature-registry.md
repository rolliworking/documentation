# Feature Registry

**Last updated:** May 27, 2026

Status legend: ✅ live & verified · 🟡 live, unverified · 🔄 in flight · ⚠️ known broken/risky · 📝 planned · ❌ abandoned

---

## Customer Portal (my.rolliworks.com/client/...)

| Feature | Status | Notes / Entry points |
|---------|--------|---------------------|
| Magic-link portal access | ✅ | `clients.token`, permanent |
| Conversation view | ✅ | `src/pages/portal/ClientConversation.tsx` |
| Progress / status display | 🟡 | Pulls from `rc-timeline-status`; band flow split in progress |
| Photo gallery | ✅ | `PhotoGallery`, `useConversationPhotos` |
| Desktop portal sidebar | 🟡 | Recently shipped |
| Documents tab | 🔄 | Built; blocked on customer ID architecture |
| Inspection view | 🟡 | `InspectionViewPage` |
| Estimate approval | 🟡 | Triggers auto-email on approval |
| Multi-job display (accordion) | 📝 | Memorial Day weekend; one accordion for desktop + mobile |

## CRM Inbox (my.rolliworks.com staff view)

| Feature | Status | Notes / Entry points |
|---------|--------|---------------------|
| Conversation inbox | ✅ | `src/pages/crm/CrmInbox.tsx`, `useInboxRows.ts` |
| Categories (tabs) | ✅ | requests / conversations / quoted / answered / archived |
| Labels (colored flags) | ✅ | yellow / red / green |
| Trigger templates | 🟡 | Move category on send |
| Unread count badge | ✅ | `get_inbox_unread_counts()` RPC |
| Shared team read state | 🟡 | One row per conversation; Mike reading clears Vienna's badge |
| Row color coding | ✅ | green = team sent last, lavender = client sent last (broke + restored 5/27) |
| Mark Unread → Conversations + bump to top | 🟡 | Updates `last_message_at`, sets category |
| Reply token display in header | ⚠️ | LOST in Test↔main merge — needs restoration |
| Conversation panel | ✅ | `src/components/crm/ConversationPanel.tsx` |
| Saved replies carousel | ✅ | `SavedRepliesCarousel` |
| Create Estimate button | 🟡 | Opens RS estimate creation, gated on `rs_customer_id` |
| Photo upload (staff) | ✅ | `PhotoUpload` |
| Save draft on conversation switch | 🔄 | In-memory; sent to RC, status unclear |
| Internal team notes (@mention) | ❌ | ABANDONED — @mike leaked to customer (RLS on team_users limited visibility); see known-issues |
| Estimate # lookup in search bar | 📝 | Option 1 (smart search bar) locked; storage approach TBD |

## Email

| Feature | Status | Notes |
|---------|--------|-------|
| Inbound handler | ✅ | `inbound-email-handler/index.ts` |
| Per-customer reply token | 🟡 | `clients.reply_token`; CC use case fixed but unverified |
| Auto-reply filter | ✅ | Auto-Submitted header check |
| Staff sender attribution | ✅ | team_users / @rolliworks.com → author_type='team' |
| Loop prevention | ✅ | *@reply.rolliworks.com dropped |
| Outbound reply notification | ✅ | `portal-send-reply-notification` via Resend |
| Welcome email | 🟡 | Layout verification pending |
| Status change notifications | 📝 | Memorial Day weekend (RW templates → Resend) |
| Email forwarding to RC | 📝 | Memorial Day weekend |

## Cross-system endpoints (on RC)

| Endpoint | Status | Purpose |
|----------|--------|---------|
| `get-rc-portal-link` | ✅ | RS/RW request portal URL by rs_customer_id or email |
| `rc-timeline-status` | 🟡 | Customer-facing status (sectioned DTO v2, flow-aware) |
| `portal-intake-router` | ✅ | Wix intake → conversation creation |
| `list-customer-approvals` | 🟡 | Inspection approvals listing |

## Customer intelligence (planned)

| Feature | Status | Notes |
|---------|--------|-------|
| LTV badges | 📝 | Lifetime spend under customer name (e.g. "12k") |
| Customer notes | 📝 | Internal notes per customer |
| Ratings / stars | 📝 | Customer rating system |

## Keyboard workflow (planned, Memorial Day weekend)

| Feature | Status | Notes |
|---------|--------|-------|
| Answered folder | ✅ | Shipped |
| Delete key → Answered | 📝 | |
| Arrow key navigation | 📝 | |
| Label filter dropdown | 📝 | Filter current tab by one label |
