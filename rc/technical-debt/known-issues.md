# Known Issues & Technical Debt

**Last updated:** May 27, 2026

Active bugs, fragile areas, and temporary hacks awaiting refactor.

---

## Active bugs

### 1. Reply token display lost in CRM header
- **Severity:** Medium
- **Symptom:** The `<reply_token>@reply.rolliworks.com` line + Copy button under client name is gone
- **Cause:** Likely the Test↔main merge (`1f26629`) kept "Test version" of `ConversationPanel.tsx`, which didn't have the reply token display block that was added on main
- **Fix:** Re-add the reply token display block to `ConversationPanel.tsx` header
- **File:** `src/components/crm/ConversationPanel.tsx`

### 2. team_users RLS breaks @mention detection
- **Severity:** Medium (caused a feature abandonment)
- **Symptom:** @mention detection matched the logged-in user's own name but not other team members'
- **Cause:** Suspected RLS policy on `team_users` restricting authenticated SELECT to own row only. Vienna querying team_users sees only Vienna → @mike doesn't match → message not flagged internal → LEAKED to customer
- **Status:** Internal notes feature ABANDONED because of this leak risk
- **Fix needed:** Review team_users SELECT policy; allow authenticated users to read full roster

### 3. Internal notes feature — ABANDONED, needs cleanup
- **Severity:** High (was leaking staff comments to customers)
- **Symptom:** `@mike` comments (from Vienna) reached the customer's email; `@vienna` correctly stayed internal
- **Decision:** Abandon the feature
- **Cleanup needed:** Disable `detectInternalMention` (return false always) OR full revert. The `is_internal` column can stay (harmless). Mark Unread bump (which works) should be kept.
- **CRITICAL:** Until removed, staff must NOT use @mentions expecting privacy

### 4. RS→RW intake push data gaps
- **Severity:** Medium (Task 2, separate chat)
- **Missing fields:** service_type (captured but not forwarded), date (always today, ignores user input), serial_number, size, material, notes, stable customer ID
- **No auth** on the endpoint (also a security gap)
- **File:** RS `src/hooks/useRolliworkingSync.ts`, RW `rollisuite-intake` edge function

### 5. Reply token CC use case unverified
- **Severity:** Medium
- **Symptom:** Sending email with reply token in CC never confirmed to route into RC
- **Handler bugs fixed:** CC scan, staff attribution, loop prevention (all in code)
- **Remaining unknown:** Whether Postmark actually delivers; no dashboard access configured

### 6. Component lookup propagation
- **Severity:** Medium
- **Symptom:** RW component lookup (used to backfill/move components through stages) doesn't update RC work queue or client page
- **Status:** Captured for status notifications work

### 7. Customer ID architecture mismatch
- **Severity:** High (blocks Documents tab + reliable cross-system linkage)
- **Detail:** RS uses RolliSuite customer UUID; RW has no rs_customer_id (email is lookup key); RC stores rs_customer_id. Lookups by wrong key fail silently (caused inspection approval URL fallback bug)
- **Status:** Needs dedicated architecture session

---

## Temporary hacks to refactor

| Hack | Location | Intended replacement |
|------|----------|---------------------|
| Mark Unread bumps via `last_message_at` update | `ConversationPanel.tsx markUnread` | Cleaner: dedicated `last_activity_at` sort field |
| HMAC token brute-force scan (O(N)) | `inbound-email-handler` | Index/lookup table at 10k+ conversations |
| Caret-delimited intake string | RS→RW push | Structured JSON with auth |
| Reply token = circular self-test | mike's own client record | Use real customer for E2E |

---

## Process / tooling debt

- **Branch divergence:** Test and main drifted heavily (16 ahead / 21 behind at one point). Lovable sometimes pushes direct to main, bypassing documented Test→main flow. Consider simplifying to main-only for a 2-person team.
- **Deploy ordering:** Lovable Publish deployed frontend WITHOUT running migration (caused 5/27 production break — silent send failure + lost colors). Migration had to be run manually in Supabase SQL editor. LESSON: ensure migrations run before/with frontend deploy.
- **Verification gap:** Many features shipped today unverified end-to-end. Verification pass recommended before adding more.
- **Backup:** Lovable provides 14-day daily backups + pre-deploy snapshots. Gaps: >14 day archive, off-platform copy, tested restore procedure, GitHub code mirror.
