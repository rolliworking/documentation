# Q-005 — Map QBO sync paths (customer / invoice / PO)

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-28
**Inputs read:**
- `apps/rs/supabase/functions/qbo-*/index.ts` (all 15 edge functions)
- `apps/rs/supabase/migrations/20260112050520_*.sql` — `qbo_tokens`, `qbo_sync_log`
- `apps/rs/supabase/migrations/20260113063729_*.sql` — `qbo_account_mappings`
- `apps/rs/supabase/migrations/20260115173159_*.sql` — `qbo_invoices`
- `apps/rs/supabase/migrations/20260110071321_*.sql` — `qbo_daily_journal`
- `apps/rs/supabase/migrations/20260110063941_*.sql` — `quickbooks_sync_log` (legacy)
- `apps/rs/src/pages/setup/QBOSyncStatusPage.tsx`, `QBOCustomerSyncPage.tsx`, `QBOInvoiceSyncPage.tsx`
- `apps/rs/src/pages/sales/SalesOrderFulfillPage.tsx`, `SalesOrderDetailPage.tsx`
- `apps/rs/src/pages/purchasing/POReceivePage.tsx`, `PurchaseOrdersList.tsx`
- `apps/rs/src/pages/pickup/PickupStationPage.tsx` (invoice-status invoke)
- `documentation/RS_CONTEXT_DOSSIER.md` §0.1, §7.7, §9.1, §12.6
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**QuickBooks Online (QBO) integration** — RolliSuite keeps RS as the operational sub-ledger while QBO remains the accounting system of record. Three distinct sync directions exist:

| Path | Direction | Business purpose |
|------|-----------|------------------|
| **Customer** | QBO ↔ RS (pull cron + manual) and RS → QBO (push on fulfill) | Keep customer master data aligned; create QBO customers when invoicing |
| **Invoice** | RS → QBO (push on SO fulfill) + QBO → RS (pull for payment/status) | Create invoices with payment links; detect paid status for pickup/ship |
| **PO receive** | RS → QBO (manual after physical receive) | Post prepaid-inventory journal entries and A/P bills when parts arrive |

OAuth tokens live in `qbo_tokens`. Background sync audit lives in `qbo_sync_log`. Invoice mirror lives in `qbo_invoices`.

---

## 2. Trigger

| Path | Trigger | Who |
|------|---------|-----|
| Customer pull (cron) | Daily ~6:00 AM UTC via `pg_cron` / external scheduler → `qbo-cron-sync` | Automated |
| Customer pull (manual) | Admin opens Setup → QBO Customers, pastes OAuth refresh token | Admin |
| Customer push | SO Fulfill or SO Detail "Push to QBO" when `customers.qbo_customer_id` is null | Staff |
| Invoice push | SO Fulfill step 3 or SO Detail push after customer + items synced | Staff |
| Invoice pull | Setup → QBO Invoices, date range selected | Admin |
| Invoice status poll | Pickup Station, Ship Station, Paid Unshipped list | Staff (per SO) |
| Payment webhook | Intuit posts Payment/Invoice events to `qbo-payment-webhook` | Intuit |
| Payment email | Inbound email to `qbo-payment-email` parser | Intuit (email fallback) |
| PO receive push | PO Receive page "Push to QBO" or Purchase Orders batch push | Staff |
| Estimate pull | Setup → QBO Estimates (manual) or cron (last 7 days) | Admin / cron |
| Account mappings | Setup → QBO Account Mappings page load | Admin |

---

## 3. Actors

| Actor | Role |
|-------|------|
| Staff (fulfillment) | Pushes customer → items → invoice chain on SO fulfill; visible toast on failure |
| Admin (setup) | Initializes OAuth via refresh token; runs manual customer/estimate/invoice pulls; views sync status |
| Intuit QBO API | OAuth2 bearer tokens; Customer, Invoice, Estimate, Item, JournalEntry, Bill endpoints |
| `qbo-cron-sync` | Scheduled worker for customer + estimate pull |
| `qbo-payment-webhook` | Marks `sales_orders.is_paid` when QBO Payment events arrive |
| `qbo-invoice-status` | Live balance check at pickup/ship gates |
| Supabase `qbo_tokens` | Stores refresh + access tokens per realm/environment |
| Supabase `qbo_sync_log` | Audit trail for background sync runs |

---

## 4. Inputs

### OAuth / credentials (all functions)

| Input | Source | Notes |
|-------|--------|-------|
| `QBO_CLIENT_ID`, `QBO_CLIENT_SECRET`, `QBO_REALM_ID` | Supabase edge function secrets | Required for all QBO API calls |
| `QBO_WEBHOOK_VERIFIER_TOKEN` | Edge secret | `qbo-payment-webhook` HMAC verification only |
| Refresh token (first connect) | Admin pastes from Intuit OAuth Playground (`RT1-...`) | Seeds `qbo_tokens` via `qbo-customer-sync` or `qbo-estimate-sync` |

### Customer sync inputs

| Input | Function | Purpose |
|-------|----------|---------|
| `refresh_token` (body) | `qbo-customer-sync` | Initial OAuth exchange on manual full sync |
| `sync_type`: `customer` \| `estimate` \| `both` | `qbo-cron-sync` | Which entities to pull |
| `environment`: `production` \| `sandbox` | All sync functions | Target QBO company |
| `customer_id` | `qbo-customer-push` | RS customer UUID to push/create in QBO |

### Invoice sync inputs

| Input | Function | Purpose |
|-------|----------|---------|
| `sales_order_id`, `qbo_customer_id` | `qbo-sales-order-push` | SO to invoice |
| `start_date`, `end_date`, `months_back`, `force_editable` | `qbo-invoice-sync` | Pull window |
| `triggered_by: "cron"` | `qbo-invoice-sync` | Skips admin auth check |
| `sales_order_id` or `qbo_invoice_id` | `qbo-invoice-status` | Payment gate check |
| `sales_order_id` | `qbo-invoice-update` | Push line edits to existing QBO invoice |
| `sales_order_id` | `qbo-so-pull` | Pull QBO header changes back to RS SO |

### PO sync inputs

| Input | Function | Purpose |
|-------|----------|---------|
| `po_id`, `po_number`, `vendor_id`, `vendor_name`, `received_date` | `qbo-po-receive` | PO header |
| `lines[]`: `part_id`, `part_number`, `description`, `qty_received`, `unit_cost`, `item_type` | `qbo-po-receive` | Receive lines for JE + Bill |

---

## 5. Steps (end-to-end flows)

### Path A: Customer sync — cron pull (suspect)

1. Scheduler invokes `qbo-cron-sync` with `{ sync_type: 'both' }` (default)
2. Function reads `qbo_tokens` for `realm_id` + `environment`; returns 400 if no token ("run manual sync first")
3. Exchanges refresh token for new access token; saves rotated refresh token immediately
4. Inserts `qbo_sync_log` row: `sync_type: 'customer'`, `status: 'started'`, `triggered_by: 'cron'`
5. Queries QBO: `SELECT * FROM Customer WHERE Active = true` (paginated, 1000/page)
6. Per customer: upserts `customers` flat fields (`qbo_customer_id`, name, email, phone, bill address line1/city/state/zip)
7. **Does not** write `customer_addresses` rows (unlike manual sync)
8. Updates log to `success` with counts, or `failed` with `error_message`
9. Repeats for estimates (last 7 days) with separate log entry
10. Updates `qbo_tokens.last_sync_at`, `last_sync_type`, `last_sync_status`

**Failure modes:** Edge function timeout on ~10k customers (per-row upsert loop); outer `catch` does not mark in-flight log as failed → stuck `started` rows; cron customer mapping diverges from manual sync.

### Path B: Customer sync — manual pull (admin)

1. Admin pastes refresh token on `QBOCustomerSyncPage` (`/setup/qbo-customers`)
2. Invokes `qbo-customer-sync` with token + environment
3. Fetches Intuit discovery document (cached 24h); exchanges token with retry/backoff
4. Paginates all QBO customers (active and inactive)
5. Upserts `customers` + `customer_addresses` (BillAddr and ShipAddr)
6. Saves tokens to `qbo_tokens`
7. **Does not** write `qbo_sync_log`

### Path C: Customer push — on SO fulfill (healthy, operator-visible)

1. `SalesOrderFulfillPage.handlePushToQBO()` checks `order.customers.qbo_customer_id`
2. If missing → invokes `qbo-customer-push` with `{ customer_id }`
3. Function creates QBO Customer or links existing on duplicate `DisplayName`
4. Sets `customers.qbo_customer_id`; returns ID to caller
5. Failure surfaces as toast — fulfill chain stops

### Path D: Invoice push — SO fulfill (healthy, primary revenue path)

1. After customer push (Path C), `qbo-item-push` syncs line-item `parts` → QBO Service items
2. `qbo-sales-order-push` creates QBO Invoice:
   - `DocNumber = so_number`
   - Line items from SO lines
   - `CustomerMemo` includes pickup verification URL
   - Optional COGS journal entry (Debit COGS, Credit Inventory Asset per `qbo_account_mappings`)
3. Sets `sales_orders.qbo_invoice_id`
4. Operator sends payment link to client

### Path E: Invoice pull + payment detection (healthy)

1. Admin runs `qbo-invoice-sync` from Setup (or cron with `triggered_by: "cron"`)
2. Inserts `qbo_sync_log` with `sync_type: "invoice"` (**schema mismatch** — see §11)
3. Pulls invoices in date window → upserts `qbo_invoices`
4. For each invoice linked to SO: updates `sales_orders` paid status, balance, tracking
5. When newly paid → may trigger shipping-info request email (`maybeRequestShippingInfo`)
6. Second pass: catch-up on older open invoices outside date window
7. Updates log with `finished_at` (**column mismatch** — migration has `completed_at`)

**Payment loopback (parallel):**
- `qbo-payment-webhook`: HMAC-verified Intuit events → fetch Payment → mark SO paid
- `qbo-invoice-status`: live `Balance` check at Pickup/Ship stations
- `qbo-payment-email`: parses Intuit payment notification emails as fallback

### Path F: PO receive → QBO (low volume, manual)

1. Staff receives PO in `POReceivePage`; clicks "Push to QBO"
2. Invokes `qbo-po-receive` with receive lines + vendor
3. Preconditions: `qbo_account_mappings` for `inventory_asset`, `grni_holding`, `cogs`, `accounts_payable`
4. **Step 1:** Journal Entry — debit Inventory/COGS, credit GRNI holding
5. **Step 2:** Bill — debit holding, credit A/P (failure logged but JE retained)
6. Stores `purchase_orders.qbo_bill_id`; JE/Bill IDs appended to `notes`
7. Operator manually applies prepaid asset to Bill in QBO UI (documented in function header)

**No cron, no `qbo_sync_log` entry.**

---

## 6. Outputs

| Output | Table / system | When |
|--------|----------------|------|
| `customers.qbo_customer_id` | `customers` | Customer push or pull |
| `customer_addresses` rows | `customer_addresses` | Manual customer pull only |
| `parts.qbo_item_id` | `parts` | `qbo-item-push` |
| `sales_orders.qbo_invoice_id`, `is_paid`, `balance_due` | `sales_orders` | Invoice push / pull / webhook |
| `qbo_invoices` mirror rows | `qbo_invoices` | Invoice pull |
| `purchase_orders.qbo_bill_id`, notes with JE/Bill IDs | `purchase_orders` | PO receive push |
| `qbo_sync_log` rows | `qbo_sync_log` | Cron customer/estimate; invoice pull (if insert succeeds) |
| `qbo_tokens` rotation | `qbo_tokens` | Every token refresh |
| QBO Invoice + payment link | QuickBooks Online | SO fulfill |
| QBO Journal Entry + Bill | QuickBooks Online | PO receive |

---

## 7. Cross-app touches

| Touch | Direction | Notes |
|-------|-----------|-------|
| RS → QBO | Push | Customer, items, invoices, PO accounting entries |
| QBO → RS | Pull | Customers, estimates, invoices, payment status |
| RS → RW | Indirect | SO fulfill calls `sendJobFinishedToRolliworking` before/alongside QBO push |
| Pickup Station | RS reads QBO | `qbo-invoice-status` before release; payment bypass if `is_paid` locally |
| RolliConnect | None direct | Customer data synced via RS tables RC reads |
| Email (Resend) | RS | Shipping-info request after invoice paid (`qbo-invoice-sync` failsafe) |

---

## 8. Edge cases

| Edge case | Behavior |
|-----------|----------|
| Duplicate QBO customer DisplayName | `qbo-customer-push` queries QBO, links existing instead of creating |
| QBO invoice deleted in QBO | `qbo-invoice-update` clears stale `qbo_invoice_id`, prompts re-push |
| Payment webhook miss | `qbo-invoice-status` poll + `qbo-payment-email` fallback |
| Pickup before QBO registers payment | Local `sales_orders.is_paid` check first; `payment_bypass_log` for known-paid bypass |
| PO Bill step fails after JE succeeds | JE kept; operator reconciles manually using IDs in PO notes |
| Cron with no prior manual token | 400 error — cron cannot bootstrap OAuth |
| ~10k active customers | Cron may hit edge function timeout |
| Invoice sync `sync_type: "invoice"` | Violates DB CHECK constraint (`customer`, `estimate` only) unless prod was altered |
| `finished_at` vs `completed_at` | Invoice sync updates non-existent column — success log update may silently fail |

---

## 9. Workarounds observed

| Workaround | Where | Why |
|------------|-------|-----|
| Customer push on fulfill instead of cron pull | `SalesOrderFulfillPage` | Operators don't rely on broken/suspect cron; push is synchronous with visible errors |
| Payment bypass at pickup/ship | `PickupStationPage`, `payment_bypass_log` | QBO payment registration latency |
| Shipping bypass auto-fill | `qbo-invoice-sync` `maybeRequestShippingInfo` | Customers with `customer_shipping_bypass` skip manual shipping form |
| Token refresh copy-pasted in ~10 functions | Each `qbo-*/index.ts` | No `_shared/qbo-auth.ts` module |
| OAuth via Playground refresh token paste | Setup pages | No full OAuth redirect flow in app |
| PO prepaid manual step in QBO | `qbo-po-receive` header comment | QBO cannot auto-apply prepaid asset |
| `quickbooks_sync_log` (legacy) | Migration only | Per-job sync log — unused by current `qbo-*` functions |

---

## 9b. Actual pg_cron inventory (added 2026-06-29 via direct pg_cron query)

The initial Q-005 discovery guessed "single daily cron for customer sync only; invoice sync manual-triggered." That was wrong. Six active pg_cron jobs (UTC):

| Cron job name | Schedule | Target function | Notes |
|---|---|---|---|
| `qbo-daily-sync` | daily 06:00 | `qbo-cron-sync` | Customer sync — confirmed broken (162 stuck 'started' rows since 2026-01-12) |
| `maintenance-hourly` | hourly on the hour | `maintenance-cron` | Scope undocumented — needs W-42 audit |
| `qbo-invoice-sync-hourly` | hourly at :15 | `qbo-invoice-sync` | Invoice sync — runs 24×/day. Silently drops all log entries due to CHECK constraint bug (PROD-FIX-001). Zero telemetry for years. |
| `rw-status-daily-5am` | daily 10:00 UTC | `rw-status-cron` | RW status push to RS |
| `vault-storage-fee-daily` | daily 08:00 | `vault-storage-cron` | Vault storage billing — needs W-42 audit for D-020 compliance |
| `daily-gold-price-update` | daily 12:00 | `gold-price-update` | Gold price refresh |

**Key implication for PROD-FIX-001:** The invoice sync CHECK constraint bug isn't just missing telemetry for occasional manual runs. Invoice sync runs 24×/day silently. Every insert has been failing since the code shipped. Thousands of log attempts lost. This elevates PROD-FIX-001 urgency.

**Discovery methodology lesson:** cron jobs live in `pg_cron.job`, not in `apps/rs/supabase/migrations/`. Future discovery tasks touching cron behavior must query `pg_cron.job` directly.

**Source:** Session 2026-06-29 pg_cron query (A-20260629-001)

---

## 10. Open questions

### Q-005-A: Is customer cron actually completing?

**Status:** answered (A-20260628-010)

### Q-005-B: Was qbo_sync_log CHECK altered in production for invoice type?

**Status:** answered (A-20260628-011)

### Q-005-C: What is the actual scheduled time for QuickBooks sync jobs?

**Status:** answered (A-20260629-001)

### Q-005-D: How many purchase orders actually made it into QuickBooks?

**Status:** answered (A-20260629-002)

---

## 11. Schema summary

### `qbo_tokens`

| Column | Type | Purpose |
|--------|------|---------|
| `realm_id` | TEXT | QBO company ID (unique with environment) |
| `refresh_token`, `access_token` | TEXT | OAuth tokens |
| `access_token_expires_at` | TIMESTAMPTZ | Refresh gate (5-min buffer in code) |
| `environment` | TEXT | `production` \| `sandbox` |
| `last_sync_at`, `last_sync_type`, `last_sync_status`, `last_sync_error` | various | Last run metadata |

### `qbo_sync_log`

| Column | Type | Purpose |
|--------|------|---------|
| `sync_type` | TEXT | CHECK: `customer`, `estimate` only in migration |
| `status` | TEXT | `started`, `success`, `failed` |
| `triggered_by` | TEXT | `manual`, `cron` |
| `records_created/updated/skipped` | INTEGER | Counts |
| `error_message` | TEXT | Failure detail |
| `intuit_tids` | TEXT[] | Intuit support transaction IDs |
| `started_at`, `completed_at` | TIMESTAMPTZ | Timing |

**⚠️ Code drift:** `qbo-invoice-sync` uses `sync_type: "invoice"` and updates `finished_at` (not in schema).

### `qbo_invoices`

Mirror of QBO invoices: `qbo_invoice_id`, `doc_number`, customer refs, amounts, `balance`, `status`, `line_items` JSONB, `tracking_number`, `is_editable`, `raw_data`, `synced_at`.

### `qbo_account_mappings`

Keys: `inventory_asset`, `cogs`, `accounts_receivable`, `accounts_payable`, `sales_retail`, `shipping_accrual`, `grni_holding`, `undeposited_funds` → QBO account IDs.

### `qbo_daily_journal`

Daily inventory/COGS rollup; `synced_to_qbo`, `qbo_journal_id`.

### Related FK columns

- `customers.qbo_customer_id`
- `parts.qbo_item_id`
- `vendors.qbo_vendor_id`
- `sales_orders.qbo_invoice_id`, `is_paid`, `balance_due`
- `purchase_orders.qbo_bill_id`

### Legacy: `quickbooks_sync_log`

Per-job sync (`job_id`, `action`, payloads) — not used by current `qbo-*` edge functions.

---

## 12. Edge function inventory (15 functions)

| Function | One-line purpose |
|----------|------------------|
| `qbo-accounts` | Fetch QBO Chart of Accounts for Setup → Account Mappings |
| `qbo-cron-sync` | Scheduled customer + estimate pull from QBO; writes `qbo_sync_log` |
| `qbo-customer-push` | Push one RS customer → QBO Customer; set `qbo_customer_id` |
| `qbo-customer-sync` | Manual full customer pull from QBO with addresses; seeds `qbo_tokens` |
| `qbo-estimate-sync` | Manual estimate pull by date range → `estimates` + lines |
| `qbo-invoice-import` | Admin historical import: QBO invoices → `qbo_invoices` + missing SOs (no UI found) |
| `qbo-invoice-status` | Live QBO invoice balance/paid check for pickup/ship gates |
| `qbo-invoice-sync` | Pull invoices by date → `qbo_invoices`; update SO paid status; shipping email |
| `qbo-invoice-update` | Push RS SO line edits → existing QBO invoice (sparse update) |
| `qbo-item-push` | Push RS `parts` → QBO Service items; set `qbo_item_id` |
| `qbo-payment-email` | Parse Intuit payment notification emails → mark SOs paid |
| `qbo-payment-webhook` | Intuit webhook (legacy + CloudEvents) → payment processing |
| `qbo-po-receive` | PO receive → QBO Journal Entry + reconciliation Bill |
| `qbo-sales-order-push` | SO fulfill → QBO Invoice + COGS JE + pickup URL in memo |
| `qbo-so-pull` | Pull QBO invoice header changes into RS SO if QBO newer |

**Note:** Dossier references 16 functions; repo contains 15 `qbo-*` directories. No `qbo-vendor-push` or similar found.

---

## 13. UI trigger map

| Function | UI location |
|----------|-------------|
| `qbo-customer-sync` | `QBOCustomerSyncPage` (`/setup/qbo-customers`) |
| `qbo-estimate-sync` | `QBOEstimateSyncPage` (`/setup/qbo-estimates`) |
| `qbo-invoice-sync` | `QBOInvoiceSyncPage` (`/setup/qbo-invoices`) |
| `qbo-accounts` | `QBOAccountMappingsPage` (`/setup/qbo-accounts`) |
| `qbo-customer-push` | `SalesOrderFulfillPage`, `SalesOrderDetailPage` |
| `qbo-item-push` | `SalesOrderFulfillPage` |
| `qbo-sales-order-push` | `SalesOrderFulfillPage`, `SalesOrderDetailPage` |
| `qbo-invoice-update` | `SalesOrderDetailPage` |
| `qbo-so-pull` | `SalesOrderDetailPage` "Sync from QuickBooks" |
| `qbo-invoice-status` | `PickupStationPage`, `ShipStationPage`, `PaidUnshippedOrdersList` |
| `qbo-po-receive` | `POReceivePage`, `PurchaseOrdersList` batch |
| *(read-only)* | `QBOSyncStatusPage` reads `qbo_tokens` + `qbo_sync_log` |

**Server-only (no UI):** `qbo-cron-sync`, `qbo-payment-webhook`, `qbo-payment-email`, `qbo-invoice-import`

---

## 14. Comparison to workflow docs / dossier

| Source | Claim | Code finding |
|--------|-------|--------------|
| RS_CONTEXT_DOSSIER §0.1 | Invoice sync healthy | **Confirmed** — synchronous fulfill chain + webhook/status poll |
| RS_CONTEXT_DOSSIER §0.1 | Customer cron suspect | **Confirmed** — divergent impl vs manual; timeout risk; stuck `started` possible |
| RS_CONTEXT_DOSSIER §0.1 | PO sync unconfirmed | **Confirmed** — manual only, low volume, Bill step can fail |
| RS_CONTEXT_DOSSIER §12.6 | pg_cron installed, schedule not in repo | **Confirmed** — UI says 6 AM UTC customers+estimates |
| Michael workflow (SO → invoice) | Push at fulfill | **Matches** `SalesOrderFulfillPage` 3-step chain |
| Michael workflow (pickup payment) | QBO latency bypass | **Deferred to Q-006** — `qbo-invoice-status` is the check; bypass is separate |

---

## 15. Sync run audit trail summary

| Mechanism | What it logs | Reliability |
|-----------|--------------|-------------|
| `qbo_sync_log` | Cron customer/estimate runs; invoice pull attempts | **Partial** — schema drift; stuck `started`; manual syncs not logged |
| `qbo_tokens.last_sync_*` | Last cron run metadata | Updated even if partial failure inside `both` sync |
| `QBOSyncStatusPage` | Displays last 20 `qbo_sync_log` rows + token status | Read-only monitor |
| Edge function `console.log` | Per-request debug in Supabase logs | Ops-only, not queryable in RS UI |
| `quickbooks_sync_log` | Legacy per-job | Unused |

---

## 16. Recommendations (preserve vs rewrite)

| Path | Verdict | Rationale |
|------|---------|-----------|
| SO → invoice push chain (`qbo-customer-push` → `qbo-item-push` → `qbo-sales-order-push`) | **Keep** | Operator-verified, business-critical, visible failures |
| Payment webhook + `qbo-invoice-status` | **Keep** | Core pickup/ship gate; extract shared auth |
| Invoice pull (`qbo-invoice-sync`) | **Keep** | Payment catch-up + shipping email; **fix** `qbo_sync_log` schema drift |
| Customer push on fulfill | **Keep** | Canonical customer→QBO path for new SOs |
| Customer cron pull | **Fix or drop** | Suspect; diverges from manual; scale risk |
| PO receive → JE/Bill | **Keep pattern** | Low volume; manual accounting bridge is acceptable |
| Token refresh duplication | **Rewrite** | Extract `_shared/qbo-auth.ts` in rebuild |
| `qbo_sync_log` | **Fix schema** | Add `invoice` to CHECK; rename `finished_at` → `completed_at` in code |

---

## 17. What "done" means (acceptance criteria check)

- ✅ All `qbo-*` edge functions inventoried with one-line purpose each (15 found; dossier says 16)
- ✅ Customer sync failure points identified: timeout, stuck `started`, no addresses in cron, divergent implementations
- ✅ Invoice sync path documented with why operator push works
- ✅ PO sync path documented as manual low-volume
- ✅ `qbo_sync_log` audit table summarized with schema drift flagged
- ✅ Three sub-sections (customer / invoice / PO) with evidence and recommendations
- ✅ Same structural sections as Q-001 (workflow, trigger, actors, inputs, steps, outputs, cross-app, edge cases, workarounds, open questions, schema)

---

_End of discovery. Q-005 complete._
