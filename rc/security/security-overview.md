# Security & Data Compliance

**Last updated:** May 27, 2026

For technical due diligence. Describes authentication, authorization, data isolation, and known security gaps.

---

## Authentication

### Staff (CRM)
- **Method:** Supabase Auth (email/password), JWT
- **Authorization table:** `team_users` (matched by email, case-insensitive `ilike`)
- **Roles:** admin (Mike), manager (Vienna). Note: an `app_role` enum exists in RS with admin/staff/manager/office/front_desk/band_room
- **Lookup:** `useAuth.tsx` resolves `team_users` row from auth email (trimmed, ilike)

### Customers (Portal)
- **Method:** Magic-link token (`clients.token`) — no password
- **Access:** URL contains token; token grants portal session
- **Lifetime:** Permanent (does not expire)

### Cross-system (server-to-server)
- **Method:** Shared service keys (`RC_SERVICE_KEY`) in request headers
- **Used by:** `rc-portal-invite`, `get-rc-portal-link`, inspection approval

---

## Row-Level Security (RLS)

### messages table
- **Authenticated SELECT:** `USING (true)` — staff see all messages (including internal)
- **Authenticated INSERT:** `WITH CHECK (true)`
- **Anon SELECT:** token-scoped — conversation must belong to client matching `current_client_token()`, AND `is_internal = false`

### conversation_read_states
- **Consolidated:** one row per conversation (team-wide read state)
- **Authenticated:** full read/write
- **Anon:** no write path (removed — customer reads must not affect team read state)

### team_users
- **KNOWN ISSUE:** RLS may restrict authenticated users to their OWN row only. This broke @mention detection (a user querying team_users sees only themselves, so @other-person doesn't match). Contributed to the abandoned internal-notes feature. Needs review — see known-issues.md

---

## Data isolation

- Customers can only access their own conversations (token-scoped RLS)
- Internal messages (`is_internal=true`) hidden from anon/portal via RLS — but feature abandoned due to detection bug
- Photos scoped by conversation

---

## Known security gaps

| Gap | Severity | Detail |
|-----|----------|--------|
| RS→RW intake push has no auth | HIGH | `rollisuite-intake` accepts `Content-Type: text/plain`, no headers — anyone could POST intake data |
| Portal token permanent + full access | MEDIUM | If `clients.token` leaks (email header, forward, log), full portal session compromised. Mitigated by separating from reply_token |
| team_users RLS over-restrictive | MEDIUM | Breaks features needing full team roster; needs policy review |
| No POSTMARK_API_TOKEN secret | LOW | Can't programmatically verify inbound delivery; manual dashboard only |

---

## PII handling

- Customer data (name, email, phone) stored in `clients` (RC) and customer records (RS)
- No payment card data stored in RC (handled in RS/external)
- Inspection photos stored in Supabase Storage, conversation-scoped

---

## Recommendations for hardening

1. Add shared-secret auth to `rollisuite-intake` (RS→RW push)
2. Review and fix team_users RLS to allow authenticated users to read full roster (needed for legitimate features) while protecting sensitive columns
3. Consider portal token rotation capability
4. Add `POSTMARK_API_TOKEN` for inbound delivery verification
5. Tested restore procedure + off-platform backup (see roadmap)
