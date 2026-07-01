# PROD-FIX-001 — Draft package for human review

> **Status:** DRAFT — do not apply until Michael approves tonight
> **Priority:** P0 (urgent — invoice sync runs 24×/day with zero telemetry)
> **Source:** PENDING.md, A-20260628-011, A-20260629-001

---

## Pre-flight check (Task 3A)

### QBO_WEBHOOK_VERIFIER_TOKEN

| Check | Result |
|-------|--------|
| Local `.env` (rollisuite) | **UNSET** |
| Production Supabase secrets | **NOT VERIFIED** — requires dashboard or `supabase secrets list` against live project |
| Code behavior when unset | `qbo-payment-webhook` **fail-open** — accepts webhooks without HMAC verification (L415–431) |

**⚠️ STOP recommendation:** Before deploying PROD-FIX-001, verify production `QBO_WEBHOOK_VERIFIER_TOKEN` is set. If unset, payment webhooks are unauthenticated — separate from invoice sync logging but same QBO surface area.

**Action for tonight:** Michael or Lovable checks Supabase Dashboard → Edge Functions → Secrets. If unset, set token from Intuit developer portal before or alongside this fix.

**Note:** This token does not block the `qbo_sync_log` migration. Proceed with migration draft review even if webhook token check is pending — but flag both for same deploy window.

---

## Problem summary

Invoice sync (`qbo-invoice-sync-hourly`, :15 past every hour UTC) returns HTTP 200 while:

1. `qbo_sync_log` INSERT fails — CHECK allows only `customer`, `estimate`; code writes `invoice`
2. Log UPDATE may match 0 rows when INSERT failed (no error surfaced)
3. Production may still run older code with `finished_at` instead of `completed_at` (local repo already fixed — verify deployed version)

---

## Draft migration (Task 3B) — DO NOT APPLY YET

**File to create in RS repo:** `supabase/migrations/20260630120000_prod_fix_001_qbo_sync_log_sync_type.sql`

Grep of all `qbo-*` edge functions for values written to `qbo_sync_log.sync_type`:

| Function | sync_type values written to qbo_sync_log |
|----------|------------------------------------------|
| `qbo-cron-sync` | `customer`, `estimate` |
| `qbo-invoice-sync` | `invoice` |

No other `qbo-*` function inserts into `qbo_sync_log`. (`qbo-invoice-status` writes `last_sync_type: 'invoice_status'` on `qbo_tokens` only — not `qbo_sync_log`.)

```sql
-- PROD-FIX-001: Allow all sync_type values written by qbo-* edge functions.
-- Previously only 'customer' and 'estimate' were allowed; invoice sync inserts failed silently.

ALTER TABLE public.qbo_sync_log
  DROP CONSTRAINT IF EXISTS qbo_sync_log_sync_type_check;

ALTER TABLE public.qbo_sync_log
  ADD CONSTRAINT qbo_sync_log_sync_type_check
  CHECK (sync_type IN ('customer', 'estimate', 'invoice'));

COMMENT ON CONSTRAINT qbo_sync_log_sync_type_check ON public.qbo_sync_log IS
  'PROD-FIX-001: must match writers in qbo-cron-sync and qbo-invoice-sync';
```

**Pre-apply verification SQL:**

```sql
-- Confirm constraint before migration
SELECT conname, pg_get_constraintdef(oid)
FROM pg_constraint
WHERE conrelid = 'public.qbo_sync_log'::regclass AND contype = 'c';

-- Confirm zero invoice rows today (expected broken state)
SELECT sync_type, status, count(*) FROM public.qbo_sync_log GROUP BY 1, 2;
```

**Note:** An identical migration file already exists in local `rolliworks/rollisuite` at `20260607120000_prod_fix_001_qbo_sync_log_sync_type.sql` — confirm whether it was ever applied to production before re-running.

---

## Draft code changes (Task 3C) — DO NOT COMMIT YET

### Change 1: Fail loudly on log INSERT failure

**File:** `apps/rs/supabase/functions/qbo-invoice-sync/index.ts` (~L439–449)

**Current (broken pattern):**

```typescript
const { data: logEntry } = await supabase
  .from("qbo_sync_log")
  .insert({
    sync_type: "invoice",
    environment,
    status: "started",
    triggered_by: isCronTrigger ? "cron" : "manual",
  })
  .select()
  .single();
// proceeds even when insert fails
```

**Draft fix:**

```typescript
const { data: logEntry, error: logInsertError } = await supabase
  .from("qbo_sync_log")
  .insert({
    sync_type: "invoice",
    environment,
    status: "started",
    triggered_by: isCronTrigger ? "cron" : "manual",
  })
  .select()
  .single();

if (logInsertError || !logEntry?.id) {
  console.error("qbo_sync_log insert failed:", logInsertError?.message);
  return new Response(
    JSON.stringify({
      error: "Failed to create sync log entry",
      detail: logInsertError?.message ?? "no log row returned",
    }),
    { status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" } }
  );
}
```

### Change 2: `finished_at` → `completed_at` on success UPDATE

**File:** `apps/rs/supabase/functions/qbo-invoice-sync/index.ts` (~L792–801)

**grep result across all `qbo-*` functions (rolliworks/rollisuite):**

| Function | finished_at | completed_at on qbo_sync_log |
|----------|-------------|------------------------------|
| qbo-invoice-sync | **none in local repo** | L797 `completed_at` ✓ |
| qbo-cron-sync | none | L184, L196, L234, L246 |
| All other qbo-* | none | n/a |

**Production may still run pre-fix build** with `finished_at`. Deploy must confirm:

```typescript
await supabase
  .from("qbo_sync_log")
  .update({
    status: "success",
    completed_at: new Date().toISOString(),  // NOT finished_at
    records_created: created,
    records_updated: updated,
  })
  .eq("id", logEntry.id);
```

### Change 3: Verify UPDATE row count + mark failed on catch

**Draft fix after success UPDATE:**

```typescript
const { error: logUpdateError, count } = await supabase
  .from("qbo_sync_log")
  .update({
    status: "success",
    completed_at: new Date().toISOString(),
    records_created: created,
    records_updated: updated,
  })
  .eq("id", logEntry.id)
  .select("id", { count: "exact", head: true });

if (logUpdateError || count === 0) {
  console.error("qbo_sync_log success update failed:", logUpdateError?.message, "count:", count);
  return new Response(
    JSON.stringify({ error: "Sync completed but log update failed", detail: logUpdateError?.message }),
    { status: 500, headers: { ...corsHeaders, "Content-Type": "application/json" } }
  );
}
```

**Draft fix in catch block** — mark log as failed when sync throws:

```typescript
} catch (error: unknown) {
  const errorMessage = error instanceof Error ? error.message : "Unknown error";
  if (logEntry?.id) {
    await supabase
      .from("qbo_sync_log")
      .update({
        status: "failed",
        error_message: errorMessage,
        completed_at: new Date().toISOString(),
      })
      .eq("id", logEntry.id);
  }
  // ... existing 500 response
}
```

---

## Verification steps (Task 3D) — run after apply

### 1. Constraint applied

```sql
SELECT pg_get_constraintdef(oid) FROM pg_constraint
WHERE conrelid = 'public.qbo_sync_log'::regclass
  AND conname = 'qbo_sync_log_sync_type_check';
-- Expect: CHECK (sync_type = ANY (ARRAY['customer'::text, 'estimate'::text, 'invoice'::text]))
```

### 2. Manual invoice sync creates durable log

1. Supabase Dashboard → Edge Functions → `qbo-invoice-sync` → Invoke with body:
   ```json
   { "environment": "production", "months_back": 1 }
   ```
2. Expect HTTP 200 with `success: true`
3. Query:
   ```sql
   SELECT id, sync_type, status, triggered_by, started_at, completed_at, records_created, records_updated
   FROM public.qbo_sync_log
   WHERE sync_type = 'invoice'
   ORDER BY started_at DESC LIMIT 5;
   ```
4. **Pass:** At least one row with `status = 'success'` and `completed_at IS NOT NULL`

### 3. Failed INSERT returns non-200 (regression guard)

Temporarily revert CHECK in staging only → invoke sync → expect HTTP 500, not silent 200.

### 4. Hourly cron telemetry (24h soak)

After next `:15` UTC cron run:

```sql
SELECT status, count(*) FROM public.qbo_sync_log
WHERE sync_type = 'invoice' AND started_at > now() - interval '24 hours'
GROUP BY status;
```

**Pass:** Rows appear with `success` or explicit `failed` — not zero rows.

### 5. QBO Sync Status UI

Setup → QBO Sync Status page should show invoice sync runs (if UI reads `qbo_sync_log`).

### 6. No regression on customer/estimate cron

```sql
SELECT sync_type, status, count(*) FROM public.qbo_sync_log
WHERE started_at > now() - interval '7 days'
GROUP BY 1, 2 ORDER BY 1, 2;
```

---

## Apply sequence (one Cursor session after approval)

1. Verify `QBO_WEBHOOK_VERIFIER_TOKEN` in production (separate concern — recommended same window)
2. Apply migration via Supabase SQL editor or `supabase db push`
3. Deploy updated `qbo-invoice-sync` edge function
4. Run manual invoke + SQL verification (steps 1–2 above)
5. Wait for one hourly cron cycle; re-query
6. Update `operational-fixes/PENDING.md` — mark PROD-FIX-001 `applied` with date + deploy SHA
7. Commit RS repo with message: `fix(qbo): PROD-FIX-001 invoice sync log telemetry`

---

## Open items for Michael tonight

- [ ] Approve migration SQL
- [ ] Approve code changes (fail-loud + completed_at + catch logging)
- [ ] Confirm production deploy path (Lovable vs direct Supabase deploy)
- [ ] Check `QBO_WEBHOOK_VERIFIER_TOKEN` in production secrets
- [ ] Confirm whether `20260607120000` migration was ever applied to prod

---

_End of draft. Do not apply without approval._
