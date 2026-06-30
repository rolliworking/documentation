# Operational fixes — pending (Lovable app)

> **What this is:** Production patches for the current Lovable-managed RolliSuite app. These are **not** rebuild work — they fix silent failures in the live system until cutover.
>
> **Status values:** `pending` | `applied` | `wont-fix`
>
> **For agents:** Do not implement here. Capture spec for Lovable or surgical deploy. Cross-reference D-020 (silent failure banned).

---

## PROD-FIX-001 — qbo-invoice-sync qbo_sync_log schema drift

**Status:** pending
**Priority:** P0
**Source:** A-20260628-011 (Q-005-B), Lovable AI audit 2026-06-28
**App:** RS (`apps/rs/supabase/functions/qbo-invoice-sync/index.ts`)
**Related decision:** D-020 Part A (silent failure banned)

### Problem

Invoice sync has **zero durable telemetry** despite returning HTTP 200. Two independent code/schema mismatches:

1. **CHECK constraint rejects INSERT** — `qbo_sync_log.sync_type` CHECK allows only `'customer'` and `'estimate'` (migration `20260112050520`, line 43). `qbo-invoice-sync` line 443 inserts `sync_type: "invoice"`. INSERT fails silently; function continues with `logEntry` undefined.

2. **Phantom column on UPDATE** — Line 797 writes `finished_at` but the table column is `completed_at` (migration line 52). UPDATE matches 0 rows when `logEntry?.id` is undefined from failed INSERT; when INSERT somehow succeeded, `finished_at` would still be ignored by Postgres.

**Evidence:** Production CHECK not altered. Zero `sync_type='invoice'` rows ever persisted. Customer sync cron separately stuck at `started` since 2026-01-12 (see A-20260628-010).

### Required fixes (Lovable)

| # | Change | File:line | Notes |
|---|--------|-----------|-------|
| 1 | Add `'invoice'` to `qbo_sync_log.sync_type` CHECK | New migration OR alter constraint | Must match all writers |
| 2 | Change `finished_at` → `completed_at` | `qbo-invoice-sync/index.ts` ~797 | Match migration schema |
| 3 | Fail loudly if log INSERT fails | `qbo-invoice-sync/index.ts` ~440–449 | Check `.error` on insert; return 500, do not proceed |
| 4 | Verify UPDATE row count | `qbo-invoice-sync/index.ts` ~793–801 | Error if 0 rows updated when logEntry expected |

### Out of scope

- CloudEvents migration — already compliant via `qbo-payment-webhook` (July 2026 deadline cleared per A-011)
- Customer sync cron rewrite — separate rebuild item under D-020; push-on-fulfill remains canonical (A-010)

### Acceptance criteria

- Manual invoice sync creates `qbo_sync_log` row with `sync_type='invoice'`, `status='success'`, `completed_at` populated
- Failed log INSERT returns non-200 and does not report success
- At least one observable row in `qbo_sync_log` for invoice sync after deploy

---

_End of pending fixes. Append-only._
