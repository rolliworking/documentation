# Q-012 — Identify RolliConnect duplicate-client root cause

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-29
**Depends on:** Q-003
**Human answers applied:** A-20260628-001 (multi-estimate/household), W-32 (related accounts)
**Inputs read:**
- `documentation/discovery/Q-003-rolliconnect-inbox-portal.md`
- `apps/rc/supabase/functions/portal-intake-router/index.ts`
- `apps/rc/supabase/functions/inbound-email-handler/index.ts`
- `apps/rs/supabase/functions/wix-intake-webhook/index.ts`, `rc-portal-invite/index.ts`
- `documentation/RC_CONTEXT_DOSSIER.md`, `RC_CONTEXT_DOSSIER2.md`
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**RC client identity** — Why RolliConnect creates duplicate `clients` rows and fragmented conversation threads, and what fix specification prevents new duplicates while enabling merge/split for historical cleanup.

---

## 2. Two distinct problems

| Problem | Symptom | Root cause |
|---------|---------|------------|
| **Duplicate clients** | Two `clients` rows for one person | Case-sensitive `email` UNIQUE + email-only lookup in `portal-intake-router` |
| **Thread fragmentation** | Multiple conversations per client (12.6% of clients) | Router **always** INSERTs new `conversations` row |

Staff often conflate these; fixes are separate but related.

---

## 3. Root causes (confirmed)

### 3.1 Case-sensitive email UNIQUE (schema + legacy data)

- PostgreSQL `text UNIQUE` on `clients.email` is case-sensitive
- Router lowercases new writes but legacy rows may be `John@example.com`
- Lookup `.eq("email", "john@example.com")` misses → second row inserted
- `backfill-rs-customer-ids` explicitly flags `case mismatch`

### 3.2 Email-only dedup; `rs_customer_id` ignored on find

- Lookup: email only → insert if miss
- RS email correction + re-invite creates second RC client (same person, new email)
- `rs_customer_id` backfilled on match but never used to find existing client

### 3.3 Alternate emails / plus-addressing

- `john+watch@gmail.com` vs `john@gmail.com` → distinct rows (business reality)

### 3.4 Conversation always created (fragmentation)

```56:93:apps/rc/supabase/functions/portal-intake-router/index.ts
// Find or create client by email only
// ...
// Always insert new conversations row
```

Returning Wix customer reuses `client_id` but gets **new thread**.

### 3.5 Inbound email — NOT a duplicate creator

- Routes by conversation HMAC token
- `inbound_sender_mismatch` flag only — no new client row (Q-003 correction)

---

## 4. Fix specification

### Phase A — Prevent new duplicates (P0)

| # | Change | File |
|---|--------|------|
| 1 | `UNIQUE (lower(email))` or `email_normalized` column + unique index | RC migration |
| 2 | One-time `UPDATE clients SET email = lower(trim(email))` | Data script |
| 3 | Lookup order: `rs_customer_id` → normalized email → phone (optional) | `portal-intake-router/index.ts` |
| 4 | Reuse open conversation for same `client_id` before INSERT | same file |

### Phase B — Merge/split UI (P1)

| # | Change | Notes |
|---|--------|-------|
| 5 | `clients.merged_into_id uuid` | Hide merged from search/inbox |
| 6 | CRM merge tool: pick survivor, reassign conversations/messages | W-16 |
| 7 | Split tool: fork conversation to new client when truly separate people | Rare |

### Phase C — Related accounts vs duplicates (W-32)

**Duplicates:** Same person, multiple rows (fix via Phase A + merge).

**Related accounts (husband/wife):** Two legitimate `customers` / `clients` rows that **share a shipment** — not merged; linked via `related_account_id` or `household_id`:

| UI behavior | Duplicate merge | Related account link |
|-------------|-----------------|----------------------|
| Same email variants | Merge into one | N/A |
| Spouse shipments | Don't merge | Show both on package card; multi-estimate link (Q-015) |
| Portal invite | One RC client per person | Optional "linked accounts" badge in CRM |

### Phase D — RS guards

- `rc-portal-invite`: check RC for existing client by `rs_customer_id` before create
- Bulk historical push (~1,150 customers) must use normalized matching

---

## 5. Acceptance test case

> RS staff corrects customer email and sends portal invite. `portal-intake-router` matches email-only; legacy case-variant row exists. Second `clients` row created. Inbox shows two clients for one person.

After fix: lookup by `rs_customer_id` finds existing row; email updated canonically; no fork.

---

## 6. Open questions

### Q-012-A: Should conversation reuse be "any open conversation" or "open conversation for same estimate/job"?

**Type:** UX
**Default:** Reuse most recent open conversation for `client_id`; staff can split manually.

---

## 7. Acceptance criteria check

- ✅ Q-003 findings confirmed + corrected (inbound email)
- ✅ Additional causes identified (rs_customer_id, fragmentation)
- ✅ Fix spec: matching + merge/split + related accounts distinction
- ✅ W-32 household handling proposed

---

_End of discovery. Q-012 complete._
