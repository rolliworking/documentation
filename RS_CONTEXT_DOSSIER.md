# RolliSuite Context Dossier

> Read-only technical context bundle for downstream model consumption (Qwen batch jobs).
> Format: dense Markdown, third-person factual voice, identifiers code-fenced for search.
> All metrics derived from live `SELECT` queries against the production Lovable Cloud
> (Supabase) database, `information_schema`, `pg_catalog`, and repository source.
> Business identity, goals, team composition, and external consumers are not derivable
> from code and are marked with `[FILL IN — …]` placeholders. See §16.

**Generated**: 2026-06-07
**Repo root**: `/dev-server` (Lovable project `939ed0e0-6220-40e3-a195-65d7c466acfe`)
**Supabase project ref**: `djbjwcoddddywkgljuja` (from `supabase/config.toml`)
**Primary domain**: `rollisuite.com` / `rollisuite.lovable.app`
**Client-facing domain**: `upload.rolliworks.com`

---

---

## 0. Verified Notes & Corrections (human-confirmed — read this first)

> This section carries product truth confirmed by Michael that the live data alone could not
> establish. It **overrides** any conflicting inference elsewhere in this dossier. It is
> maintained by hand and survives regeneration of the technical sections below. When a machine
> regenerates the dossier, re-apply this block.

### 0.1 QBO sync — actual status (corrects §12 QBO findings)
- **Sales Order → QBO (invoice) sync is HEALTHY and continuously verified.** Pushing an SO from
  RS to QBO generates the invoice and the client payment link in one continuous, operator-run
  flow. A failure cannot hide here — no invoice would mean no payment link to send — and no
  failures have occurred. Treat SO/invoice→QBO as working.
- **Customer sync (background cron) is the open suspect.** The dossier found the daily customer
  sync rows sitting at `started` with no terminal status. Because this is an unattended cron
  (unlike the interactive SO flow), a silent failure would go unnoticed. **Intuit changed QBO
  payload requirements around the same period**, which plausibly broke the customer-object cron
  while leaving the invoice path intact.
  - **Resolution check (do this):** open QBO and confirm whether customers created in RS over
    the last ~30 days actually appear. Present → cron completes fine, `started` is just a
    logging quirk, close it. Missing → cron is genuinely stuck; backfill + fix the payload.
- **Purchase Order → QBO (bill) sync: UNCONFIRMED.** Low volume (~80 POs lifetime). Verify
  whether POs are reaching QBO as bills.

### 0.2 Jobs single-status finding (corrects/confirms §7.3, §12)
- The finding that all `jobs` rows sit at `status='intake'` is expected: **the job lifecycle
  is carried in RolliWorking (RW), not RS.** RS opens the job; RW transitions it. This is a
  real cross-app boundary and must be documented in any rebuild brief — RS is not the source of
  truth for job *status*. (`job_status_history` / `job_activity_log` tables exist but the
  active workflow lives in RW.)

### 0.3 Brand tagging sparseness (corrects §12 business metrics)
- Only ~90 of 6,143 SO lines carry a joined watch brand. **OPEN:** confirm whether this is
  because most SO lines are non-watch (parts/supplies) or because brand tagging on `watches`
  is manual and lagging. Matters for any analytics that key off brand/reference.


## Table of Contents

1. [Front Matter](#1-front-matter)
2. [Identity & Goals](#2-identity--goals)
3. [System Overview](#3-system-overview)
4. [Module Map](#4-module-map)
5. [Database Catalog](#5-database-catalog)
6. [Roles, Auth & Access Control](#6-roles-auth--access-control)
7. [Data Model — Domain Groupings](#7-data-model--domain-groupings)
8. [Database Functions & Triggers](#8-database-functions--triggers)
9. [Edge Functions](#9-edge-functions)
10. [External Integrations](#10-external-integrations)
11. [Who Depends On RolliSuite](#11-who-depends-on-rollisuite)
12. [Live Usage Data](#12-live-usage-data)
13. [Dependency Inventory](#13-dependency-inventory)
14. [Stability Rules & Frozen Files](#14-stability-rules--frozen-files)
15. [Glossary](#15-glossary)
16. [Open Questions — `[FILL IN]` Index](#16-open-questions--fill-in-index)
17. [End-to-End Workflows (restored)](#17-end-to-end-workflows-restored-detail)
18. [Critical Constraints — Core Rules (restored)](#18-critical-constraints--core-rules-verbatim-restored)

*Section 0 (Verified Notes & Corrections) appears above — human-confirmed truth that overrides inferences below.*

---

## 1. Front Matter

| Field | Value |
|---|---|
| Project name | RolliSuite (RS) |
| Repo type | Lovable / Vite / React SPA + Supabase backend |
| Code stack | React 18.3, Vite 5.4, TypeScript 5.8, Tailwind 3.4, shadcn/ui (Radix), TanStack Query 5, React Router 6.30 |
| Backend | Supabase (Postgres 15, RLS, Edge Functions Deno, Storage, Auth) — "Lovable Cloud" |
| Companion repo | `app/apps/rollisuite-api` (Fastify) + `app/apps/rollisuite-web` (Vite) deployed via Railway / Nixpacks |
| Public surfaces | `/kiosk`, `/book`, `/pickup`, `/pickup-pass`, `/request-label/:estimateId`, `/return-shipping-info/:token`, `/shipping-bypass/:token`, `/upload-photos/:token`, `/go/:code` |
| Staff-only route | `/staff/parts-lookup` (no sidebar) |
| Audience | Office and shop floor staff — non-public application |

---

## 2. Identity & Goals

> The following six fields cannot be derived from source. They are explicitly left for
> the project owner to fill in. Everything else in this dossier is derived.

- **Legal entity / company name**: `[FILL IN — legal entity that operates RolliSuite]`
- **Physical location(s) / shop address(es)**: `[FILL IN — primary shop address and any additional locations]`
- **Business description (one paragraph)**: `[FILL IN — what the business does, in plain language]`
- **Primary customer segments**: `[FILL IN — e.g. retail watch repair clients, B2B trade accounts, wholesale, etc.]`
- **Strategic goals for next 6–12 months**: `[FILL IN — north-star goals the software should support]`
- **Team size and seat count per role**: `[FILL IN — headcount per role: admin / manager / office / front_desk / staff / band_room]`

### Derivable identity facts

- Operator email seeded as `admin` via DB trigger `set_admin_for_owner`: `mike@rolliworks.com`
- Branded entity referenced in code and integration: **Rolliworks** (also `RolliWorking`, the watchmaker companion app)
- Email sender identity (per `mem://auth/email-communications-identity`): `noreply@quotes.rolliworks.com` (Resend)
- Custom domains configured: `rollisuite.com` (primary), `upload.rolliworks.com` (client-facing portal routes)

---

## 3. System Overview

### 3.1 Frontend stack

```
React 18.3.1            UI framework
Vite 5.4.19             Bundler + dev server
TypeScript 5.8.3        Language
Tailwind CSS 3.4.17     Styling
shadcn/ui (Radix)       Component primitives (28 @radix-ui packages)
TanStack Query 5.83     Server-state cache
React Router 6.30       Routing
React Hook Form 7.61    Form state
Zod 3.25                Schema validation
date-fns 3.6 / date-fns-tz 3.2
Recharts 2.15           Charting
Lucide React 0.462      Icons
Sonner 1.7              Toasts
```

### 3.2 Specialty libraries

```
pdfjs-dist 4.8           PDF parsing (shipping audit, tracking extraction)
jspdf 2.5.2              PDF generation (estimates, reports)
html2canvas 1.4.1        Canvas rasterization
bwip-js 4.8              Barcode rendering
jsbarcode 3.12 / qrcode.react 4.2
html5-qrcode 2.3         Camera-based QR scanning
@dnd-kit/core 6.3        Drag-and-drop (cycle counts, sortable lines)
jszip 3.10               ZIP packing
input-otp 1.4            PIN entry UI
```

### 3.3 Backend (Supabase / Lovable Cloud)

```
Postgres                 15 (managed)
Extensions               pg_cron 1.6.4, pg_net 0.19.5, pg_stat_statements 1.11,
                         pg_trgm 1.6, pgcrypto 1.3, plpgsql 1.0,
                         supabase_vault 0.3.1, uuid-ossp 1.1
Auth                     Supabase Auth (email/password, magic link, MFA TOTP+SMS, PIN)
Storage                  Supabase Storage buckets + Cloudflare R2 for inspection photos
Edge Functions           74 Deno functions under supabase/functions/
RLS                      Enabled on 118/118 public base tables (100% coverage)
Policies                 332 row-level policies across public schema
```

### 3.4 Topology

```
                         ┌──────────────────────────────┐
                         │  Browser (office / shop)     │
                         │  React SPA — vite build      │
                         └──────────────┬───────────────┘
                                        │ HTTPS
              ┌─────────────────────────┼────────────────────────────┐
              │                         │                            │
   ┌──────────▼─────────┐   ┌───────────▼────────────┐   ┌───────────▼────────┐
   │ Supabase REST/RT   │   │ Supabase Edge Funcs    │   │ Supabase Storage   │
   │ (PostgREST + RLS)  │   │ (Deno, 74 functions)   │   │ + Cloudflare R2    │
   └──────────┬─────────┘   └──┬─────────────────────┘   └────────────────────┘
              │                │
   ┌──────────▼────────────────▼──────────────┐
   │ Postgres 15 (118 tables, 87 functions,   │
   │ 68 triggers, 332 RLS policies)           │
   └─────┬──────────────────────────────┬─────┘
         │                              │
   ┌─────▼──────┐  ┌──────────────┐  ┌──▼────────────────┐
   │ QuickBooks │  │ Resend (mail)│  │ RolliWorking      │
   │ Online API │  │ Twilio (SMS) │  │ (watchmaker app)  │
   └────────────┘  └──────────────┘  └───────────────────┘
                          │
                  ┌───────▼────────────┐
                  │ Lovable AI Gateway │
                  │ (OpenAI / Gemini)  │
                  └────────────────────┘
```

### 3.5 Companion monorepo (`app/`)

Files present: `app/apps/rollisuite-api/{nixpacks.toml, railway.json, .env.railway.api.example}`,
`app/apps/rollisuite-web/{nixpacks.toml, railway.json, .env.railway.web.example}`.
Documentation: `docs/RAILWAY_DEPLOYMENT.md`, `docs/RAILWAY_SETUP_CHECKLIST.md`.
Production state of the companion: `[FILL IN — current production status of the Railway monorepo: live / staging-only / not yet deployed]`.

---

## 4. Module Map

Counts: 109 page `.tsx` files under `src/pages/`, 158 component `.tsx` files under `src/components/`, 31 hooks under `src/hooks/`.

### 4.1 Estimates
- Routes: `/estimates`, `/estimates/new`, `/estimates/:id`, `/estimates/templates`, `/estimates/email-templates`, `/estimates/waitlist`
- Pages: `EstimatesPage.tsx`, `CreateEstimatePage.tsx`, `EstimateDetailPage.tsx` (67 KB, high-risk), `EstimateTemplatesPage.tsx`, `WaitlistPage.tsx`, `CustomersPage.tsx`, `CustomerDetailPage.tsx`
- Hooks: `useEstimates.ts`, `useEstimateTemplates.ts`, `useLineItems.ts`
- Tables: `estimates`, `estimate_line_items`, `estimate_templates`, `estimate_template_lines`, `estimate_attachments`, `waitlist`
- Edge fns: `send-estimate-email`, `qbo-estimate-sync`
- Permission gate: open to all authenticated users; deletion guarded elsewhere
- Memory refs: `mem://features/estimates/*`

### 4.2 Intake
- Routes: `/intake/leads`, `/intake/email`, `/intake/receive-watch`, `/intake/history`, `/intake/receive-packages`, `/intake/shipping-labels`, `/intake/ifs-label-generator`, public `/kiosk`
- Pages: `IntakeLeadsPage.tsx`, `IntakeEmailPage.tsx`, `ClientWatchEntryPage.tsx`, `WatchIntakeHistoryPage.tsx`, `ReceivePackagesPage.tsx`, `ShippingLabelRequestsPage.tsx`, `IFSLabelGeneratorPage.tsx`, `public/KioskPage.tsx`
- Tables: `intake_leads`, `package_arrival_scans`, `package_scan_logs`, `client_property`, `shipping_info_submissions`, `received_webhook_events`
- Edge fns: `wix-intake-webhook` (anon), `parts-webhook` (anon), `scan-shipping-label`, `send-receive-report`, `send-shipping-info-request`, `request-shipping-label`
- Permissions: `canAccessIntake`, `canAccessReceiveWatch`
- Memory refs: `mem://features/intake/*`

### 4.3 Inventory
- Routes: `/inventory/parts`, `/inventory/parts/categories`, `/inventory/parts/import-*`, `/inventory/client-property`, `/inventory/calibers`, `/inventory/reorder-alerts`, `/inventory/low-stock`, `/inventory/cycle-count*`, `/inventory/quick-move`, `/inventory/reference-library`, `/inventory/products`
- Pages: `PartsPage.tsx` (65 KB, high-risk), `PartCategoriesPage.tsx`, `ClientPropertyPage.tsx`, `CalibersPage.tsx`, `ReorderAlertsPage.tsx`, `LowStockReportPage.tsx`, `CycleCountPage.tsx`, `CycleCountScannerPage.tsx`, `CycleCountiPadPage.tsx`, `QuickMovePage.tsx`, `ReferenceLibraryPage.tsx`
- Tables: `parts`, `part_aliases`, `part_categories`, `part_vendor_prices`, `vendor_parts`, `inventory_stock`, `inventory_moves`, `inventory_adjustments`, `inventory_transfers`, `transfer_lines`, `cycle_counts`, `cycle_count_lines`, `client_property`, `calibers`, `caliber_parts`, `caliber_vendors`, `caliber_sample_sheets`, `bracelet_models`, `reference_bracelets`, `model_references`, `bins`, `locations`, `stores`
- Edge fns: `parts-pricing`, `parts-webhook`, `gold-price-update` (cron)
- Permission gate: `canAccessInventory`
- Memory refs: `mem://features/inventory/*`

### 4.4 Purchasing
- Routes: `/purchasing/orders`, `/purchasing/drafts`, `/purchasing/issued`, `/purchasing/vendors`, `/purchasing/vendors/{import,pricing-import,parts-import}`, `/purchasing/shop-work-orders`, `/purchasing/disassembly`, `/purchasing/receiving`
- Pages: `PurchaseOrdersPage.tsx`, `VendorsPage.tsx`, `VendorImportPage.tsx`, `VendorPricingImportPage.tsx`, `VendorPartsImportPage.tsx`, `ShopWorkOrdersPage.tsx`, `DisassemblyOrdersPage.tsx`, `POReceivePage.tsx`
- Tables: `purchase_orders`, `po_lines`, `po_receipts`, `po_receipt_lines`, `vendors`, `vendor_parts`, `vendor_price_history`, `part_vendor_prices`, `shop_work_orders`, `shop_work_order_lines`, `swo_lines`, `service_work_orders`, `reorder_suggestions`
- Sequences: `po_number_seq`, `swo_number_seq`
- Edge fns: `qbo-po-receive`, `send-po-email`
- Permission gate: `canAccessPurchasing`
- Memory refs: `mem://features/purchasing/*`

### 4.5 Sales Orders
- Routes: `/sales/orders`, `/sales/orders/:id`, `/sales/orders/:id/fulfill`, `/sales/pickup-station`, `/sales/ship-station`
- Pages: `SalesOrdersPage.tsx`, `SalesOrderDetailPage.tsx` (34 KB, high-risk), `SalesOrderFulfillPage.tsx`, `PickupStationPage.tsx`, `ShipStationPage.tsx`
- Tables: `sales_orders`, `so_lines`, `shipments`, `shipment_logs`, `shipping_labels`, `shipping_info_submissions`, `customer_shipping_bypass`, `pickup_sessions`, `pickup_verification_codes`, `payment_bypass_log`, `vault_storage_fees`
- Edge fns: `qbo-sales-order-push`, `qbo-so-pull`, `qbo-invoice-sync`, `qbo-invoice-update`, `qbo-invoice-status`, `qbo-payment-webhook`, `qbo-payment-email`, `send-fedex-pickup-notice`, `send-pickup-pass`, `send-pickup-reminder`, `send-pickup-reminder-sms`, `send-payment-reminder`, `send-payment-reminder-sms`, `send-bypass-optin`, `shipping-bypass-optin`, `return-shipping-info`, `vault-storage-cron` (cron)
- Memory refs: `mem://features/sales/*`, `mem://features/pickup/*`, `mem://features/shipping/*`

### 4.6 Pickup Station (public verification surface)
- Routes (public): `/pickup`, `/pickup-pass`
- Pages: `public/PickupVerificationPage.tsx`, `public/ClientPickupPassPage.tsx`
- Tables: `pickup_verification_codes`, `pickup_sessions`
- DB functions: `generate_pickup_code()`, `get_pickup_pass(p_code text)` (SECURITY DEFINER, anon-callable RPC)
- Edge fns: `send-pickup-pass`, `decrypt-proxy-id`, `encrypt-proxy-id`
- Memory refs: `mem://features/pickup/pickup-ticket-system`, `mem://features/pickup/pickup-station-workflow`, `mem://features/pickup/proxy-id-security`

### 4.7 Ship Station
- Routes: `/sales/ship-station`
- Page: `ShipStationPage.tsx`
- Component: `sales/PaidUnshippedOrdersList.tsx`
- Tables: `shipping_labels`, `shipping_info_submissions`, `shipments`, `shipment_logs`, `package_scan_logs`
- Edge fns: `scan-shipping-label`, `send-shipping-label-email`, `request-shipping-label`, `send-fedex-pickup-notice`
- Memory refs: `mem://features/shipping/*`

### 4.8 Labels
- Routes: `/labels`, `/labels/designer`
- Pages: `Labels2Page.tsx`, `labels/LabelDesignerPage.tsx`
- Components: `BarcodeLabel.tsx`, `BinLabelTab.tsx`, `Code128Barcode.tsx`, `CreateLabelDialog.tsx`, `IntakeLabelWizard.tsx`, `LabelPreview.tsx`, `LabelPrintDialog.tsx`, `QRLabelSearchInputs.tsx`, `SequentialLabelTab.tsx`
- Libraries: `zpl-generator.ts`, `zpl-labels.ts`, `part-label-print.ts`, `print-window.ts`
- Tables: `label_prints`, `pending_print_jobs`, `print_queue`
- Edge fns: `rw-print-labels`
- Memory refs: `mem://features/labels/asset-label-spec`, `mem://infrastructure/print-window-architecture`

### 4.9 Reports
- Routes: `/reports`, `/reports/presets`, `/reports/analytics`, `/reports/appraisals`, `/reports/band-funnel`, `/reports/inventory`, `/reports/stale-inventory`, `/reports/inventory-by-category`, `/reports/department-performance`, `/reports/parts-sales-category`, `/reports/band-room-activity`, `/reports/band-room-jobs`, `/reports/process-analytics`, `/reports/watchmaker-revenue`, `/reports/shipping-audit`, `/reports/supplies`, `/reports/parts-consumed`, `/reports/department-revenue`, `/reports/performance/new`, `/reports/performance/:id`, `/reports/performance/:id/preview`
- Pages: (21 page files under `src/pages/reports/`)
- Tables: `report_presets`, `appraisals`, `appraisal_templates`, `appraisal_field_options`, `performance_verification_reports`, `performance_verification_grades`, `supplies`
- Hooks: `useAppraisals.ts`, `usePerformanceReport.ts`, `useReportPresets.ts`, `useToleranceSheetExtraction.ts`
- Library: `performance-report-logic.ts`
- Permission gate: `canAccessReportAnalytics`, `canAccessBandRoom` for band-room reports
- Memory refs: `mem://features/reports/*`

### 4.10 Inspection
- Routes: `/inspection/capture/:id`, `/inspection/library`, `/inspection/library/:id`, `/inspection/import`
- Pages: `InspectionCapturePage.tsx`, `InspectionLibraryPage.tsx`, `InspectionDetailPage.tsx`, `InspectionImportPage.tsx`
- Tables: `inspections`, `inspection_requests`, `inspection_photos`
- Storage: Cloudflare R2 (`get-photo`, `upload-photo`, `upload-client-photos` edge fns)
- Edge fns: `extract-tolerance-sheet`, `compare-signatures` (Lovable AI gateway)
- Permission gate: `canAccessInspections`
- Memory refs: `mem://features/inspection/*`, `mem://auth/inspection-photo-access`

### 4.11 Jobs (light surface)
- Routes: `/jobs/shop-time`
- Pages: `jobs/ShopTimePage.tsx`, plus `CreateJob.tsx` and `JobDetail.tsx` at root
- Tables: `jobs`, `job_status_history`, `job_activity_log`, `job_queue`, `shop_time_entries`, `watchmakers`, `watchmaker_parts`
- DB functions: `generate_job_id()`, `get_next_job_id()`, `job_id_exists()`, `claim_next_job()`, `log_job_status_change()` (trigger), `get_job_profile(p_estimate_id uuid)` (large composite payload for downstream sync)
- Edge fns: `rollisuite-job-finished`, `job-status-update`, `rw-job-status`, `rw-watchmaker-assignment`
- Memory refs: `mem://features/repair/turnaround-targets`

### 4.12 Setup / Admin
- Routes: `/setup/{users,locations,email-templates,qbo-customers,qbo-estimates,qbo-status,qbo-invoices,qbo-accounts,security,company,import-export,admin-log,arrival-log,custody-log,scheduling,pickup-camera,caliber-sample-sheets}`
- Pages: 17 pages under `src/pages/setup/`
- Tables: `settings`, `role_permissions`, `user_roles`, `user_login_pins`, `user_sessions`, `invitations`, `audit_log`, `message_templates`, `scheduling_settings`, `scheduling_closed_dates`, `scheduling_notes`, `scheduling_type_rules`, `camera_settings`, `caliber_sample_sheets`, `qbo_account_mappings`, `qbo_tokens`, `qbo_sync_log`, `quickbooks_sync_log`, `qbo_invoices`
- Permission gate: `canAccessSetup` (admin/owner only)

### 4.13 Public / Client-Facing Surfaces
- All on `upload.rolliworks.com` per `mem://architecture/client-facing-domain-policy`
- Routes: `/kiosk`, `/book`, `/pickup`, `/pickup-pass`, `/request-label/:estimateId`, `/return-shipping-info/:token`, `/shipping-bypass/:token`, `/upload-photos/:token`, `/go/:code`
- Pages: 9 files under `src/pages/public/`
- Anon-callable RPCs: `get_pickup_pass`, `get_invitation_by_token`, `get_photo_request_by_token`
- Edge fns with `verify_jwt = false` (44 functions, see §9): all public-token webhooks, QBO callbacks, Wix intake

### 4.14 Dashboard & Daily Hit List
- Routes: `/dashboard`, `/daily-hit-list`, `/workspace`
- Pages: `DashboardNew.tsx`, `DailyHitListPage.tsx`
- Components: `workspace/DashboardTab.tsx`, `workspace/CustomersListTab.tsx`, `workspace/PartsListTab.tsx`, `workspace/ReorderDraftsTab.tsx`, `workspace/VendorDraftsTab.tsx`, `workspace/WorkspaceContent.tsx`, `workspace/TabBar.tsx`
- Permission gate: `canAccessDashboard`
- Memory refs: `mem://features/sales/daily-hit-list`, `mem://features/reports/dashboard-metrics`, `mem://integrations/rolliworking-hit-list-integration`

### 4.15 Customers
- Routes: `/customers`, `/customers/:id`
- Pages: `estimates/CustomersPage.tsx`, `estimates/CustomerDetailPage.tsx`
- Components: `customers/CustomerSidePanel.tsx` (46 KB, high-risk), `CustomerHistory.tsx`, `CustomerSelector.tsx`, `DuplicateCustomerDialog.tsx`, `SendPortalInviteButton.tsx`, `ShippingBypassDialog.tsx`, `CustomerCSVInstructionsDialog.tsx`
- Tables: `customers`, `customer_addresses`, `customer_communication_permissions`, `customer_shipping_bypass`, `watches`
- DB function: `update_customer_normalized_fields()` trigger (email/phone normalization + display_name sync)
- Memory refs: `mem://features/customers/*`

---

## 5. Database Catalog

### 5.1 Object counts

```sql
-- Tables
SELECT count(*) FROM information_schema.tables
WHERE table_schema='public' AND table_type='BASE TABLE';
-- Views
SELECT count(*) FROM information_schema.views WHERE table_schema='public';
-- Functions
SELECT count(*) FROM pg_proc p JOIN pg_namespace n ON n.oid=p.pronamespace
WHERE n.nspname='public';
-- Triggers (excluding internal)
SELECT count(*) FROM information_schema.triggers WHERE trigger_schema='public';
-- RLS policies
SELECT count(*) FROM pg_policies WHERE schemaname='public';
-- Tables with RLS enabled
SELECT count(*) FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE n.nspname='public' AND c.relkind='r' AND c.relrowsecurity=true;
```

| Object | Count |
|---|---:|
| Base tables (`public`) | 118 |
| Views (`public`) | 0 |
| Functions (`public`) | 87 (incl. `pg_trgm` operator funcs) |
| Triggers (user-defined) | 68 |
| RLS policies | 332 |
| Tables with RLS enabled | 118 (100%) |
| Sequences | 2 (`po_number_seq`, `swo_number_seq`) |
| Extensions | 8 |
| Enums | 16 |

### 5.2 All base tables

```sql
SELECT table_name FROM information_schema.tables
WHERE table_schema='public' AND table_type='BASE TABLE'
ORDER BY table_name;
```

```
api_rate_limits, appointments, appraisal_field_options, appraisal_templates, appraisals,
attachments, audit_log, bins, bom_headers, bom_lines, bracelet_models, cache_store,
caliber_parts, caliber_sample_sheets, caliber_vendors, calibers, camera_settings,
client_photo_requests, client_photo_submissions, client_property, csv_import_staging,
customer_addresses, customer_communication_permissions, customer_shipping_bypass,
customers, cycle_count_lines, cycle_counts, estimate_attachments, estimate_line_items,
estimate_template_lines, estimate_templates, estimates, inspection_photos,
inspection_requests, inspections, intake_leads, inventory_adjustments, inventory_moves,
inventory_stock, inventory_transfers, invitations, job_activity_log, job_queue,
job_status_history, jobs, label_prints, line_items, locations, message_templates,
model_references, outbound_events, package_arrival_scans, package_scan_logs,
part_aliases, part_categories, part_vendor_prices, parts, payment_bypass_log,
pending_print_jobs, performance_verification_grades, performance_verification_reports,
pickup_sessions, pickup_verification_codes, po_lines, po_receipt_lines, po_receipts,
pressure_tests, print_queue, profiles, purchase_orders, qbo_account_mappings,
qbo_daily_journal, qbo_invoices, qbo_sync_log, qbo_tokens, quickbooks_sync_log,
received_webhook_events, reference_bracelets, reorder_suggestions, report_presets,
role_permissions, rw_hit_list_items, sales_orders, scheduling_closed_dates,
scheduling_notes, scheduling_settings, scheduling_type_rules, service_categories,
service_subcategories, service_work_orders, settings, shipment_logs, shipments,
shipping_info_submissions, shipping_labels, shipping_rates, shop_time_entries,
shop_work_order_lines, shop_work_orders, short_urls, so_lines, stores, supplies,
swo_lines, tc_acceptances, timing_tests, transfer_lines, user_login_pins, user_roles,
user_sessions, vault_storage_fees, vendor_parts, vendor_price_history, vendors,
waitlist, watches, watchmaker_parts, watchmakers
```

### 5.3 Top tables by size

```sql
SELECT c.relname, pg_size_pretty(pg_total_relation_size(c.oid))
FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE n.nspname='public' AND c.relkind='r'
ORDER BY pg_total_relation_size(c.oid) DESC LIMIT 30;
```

| Table | Size |
|---|---:|
| `qbo_invoices` | 38 MB |
| `estimate_line_items` | 10 MB |
| `customers` | 8.4 MB |
| `parts` | 4.5 MB |
| `rw_hit_list_items` | 3.3 MB |
| `so_lines` | 2.5 MB |
| `inspection_photos` | 2.4 MB |
| `estimates` | 2.0 MB |
| `intake_leads` | 1.9 MB |
| `jobs` | 896 kB |
| `inventory_stock` | 816 kB |
| `watches` | 696 kB |
| `estimate_template_lines` | 688 kB |
| `shipments` | 656 kB |
| `part_vendor_prices` | 624 kB |
| `package_scan_logs` | 624 kB |
| `sales_orders` | 584 kB |
| `shipment_logs` | 560 kB |
| `vendor_parts` | 496 kB |
| `job_activity_log` | 448 kB |
| `qbo_tokens` | 432 kB |
| `received_webhook_events` | 432 kB |
| `shipping_info_submissions` | 416 kB |
| `client_photo_submissions` | 376 kB |
| `tc_acceptances` | 376 kB |
| `package_arrival_scans` | 368 kB |
| `model_references` | 360 kB |
| `shipping_labels` | 336 kB |
| `inspection_requests` | 312 kB |
| `customer_addresses` | 304 kB |

### 5.4 Per-table RLS policy counts

```sql
SELECT c.relname, COUNT(*) FROM pg_policies p
JOIN pg_class c ON c.relname=p.tablename
JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE p.schemaname='public' AND n.nspname='public'
GROUP BY c.relname ORDER BY c.relname;
```

Sample (full list 118 rows; tables not listed have 0 policies despite RLS-enabled, meaning denied-by-default):

```
appointments=4  appraisal_templates=3  appraisals=4  attachments=4  audit_log=2
client_photo_requests=4  client_property=4  customer_addresses=4  customers=4
estimate_line_items=4  estimates=4  intake_leads=4  inspection_requests=4
jobs=4  parts=4  profiles=6  purchase_orders=4  sales_orders=5  user_roles=2
watches=4  ... (see /tmp dump for full)
```

### 5.5 Enums

```sql
SELECT t.typname, string_agg(e.enumlabel, ',' ORDER BY e.enumsortorder)
FROM pg_type t JOIN pg_enum e ON e.enumtypid=t.oid
JOIN pg_namespace n ON n.oid=t.typnamespace
WHERE n.nspname='public' GROUP BY t.typname ORDER BY t.typname;
```

| Enum | Values |
|---|---|
| `adjustment_type` | cycle_count, shrinkage, damage, correction, disassembly, assembly |
| `app_role` | admin, staff, manager, office, front_desk, band_room |
| `custody_status` | in_custody, released |
| `cycle_count_status` | draft, in_progress, submitted, approved, posted |
| `estimate_status` | draft, sent, converted, expired, declined |
| `item_type` | inventory, non_inventory, service, client_property, assembly, client_watch, shipping, service_marker |
| `job_status` | intake, in_review, awaiting_customer_approval, approved, in_service, testing, ready_to_ship, closed |
| `outbound_event_status` | pending, sent, failed |
| `po_status` | draft, issued, partial_received, received, cancelled |
| `priority_level` | low, normal, high, urgent |
| `received_webhook_status` | received, processed, duplicate, failed |
| `service_type` | watch_service, bracelet_service, other_service, warranty_service |
| `simple_job_status` | estimate, on_hand, finished |
| `so_status` | draft, open, partial_fulfilled, fulfilled, shipped, picked_up, cancelled |
| `transfer_status` | pending, in_transit, completed, cancelled |
| `uom_type` | ea, hr, box, set, lb, oz, ft, in |

### 5.6 Extensions

```sql
SELECT extname, extversion FROM pg_extension ORDER BY extname;
```

`pg_cron 1.6.4` · `pg_net 0.19.5` · `pg_stat_statements 1.11` · `pg_trgm 1.6` ·
`pgcrypto 1.3` · `plpgsql 1.0` · `supabase_vault 0.3.1` · `uuid-ossp 1.1`

### 5.7 Sequences

```sql
SELECT sequence_name FROM information_schema.sequences
WHERE sequence_schema='public' ORDER BY sequence_name;
```

`po_number_seq`, `swo_number_seq` (other numbering — estimates, SOs, jobs, transfers,
moves, cycle counts, appraisals — uses `settings.next_*` columns or per-table MAX
queries inside generator functions, not sequences. See §8.)

---

## 6. Roles, Auth & Access Control

### 6.1 Role enum (`public.app_role`)

`admin`, `manager`, `office`, `front_desk`, `staff`, `band_room`

### 6.2 Role-to-permission matrix (defaults from `src/types/database.ts`)

Permissions are also overridable per-role in the `role_permissions` table; the hook
`useRolePermissions.ts` reads the DB first, then falls back to these defaults. `office`
and `staff` map to DB role `team`; `front_desk` and `band_room` stay as-is.

| Permission | admin | manager | office | front_desk | staff | band_room |
|---|:-:|:-:|:-:|:-:|:-:|:-:|
| `canDeleteJobs` | ✓ |  |  |  |  |  |
| `canManageUsers` | ✓ |  |  |  |  |  |
| `canManageSettings` | ✓ |  |  |  |  |  |
| `canAccessSetup` | ✓ |  |  |  |  |  |
| `canExportCSV` | ✓ | ✓ |  |  |  |  |
| `canEditTestsAnytime` | ✓ | ✓ |  |  |  |  |
| `canEditTestsWithin24Hours` | ✓ | ✓ | ✓ |  | ✓ |  |
| `canAccessIntake` | ✓ | ✓ | ✓ | ✓ |  |  |
| `canAccessInspections` | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| `canAccessReceiveWatch` | ✓ | ✓ | ✓ | ✓ |  |  |
| `canAccessReportAnalytics` | ✓ | ✓ |  |  |  |  |
| `canAccessPurchasing` | ✓ | ✓ |  |  |  |  |
| `canAccessInventory` | ✓ | ✓ |  |  |  |  |
| `canAccessDashboard` | ✓ | ✓ |  |  |  |  |
| `canAccessCustomers` | ✓ | ✓ | ✓ | ✓ |  |  |
| `canEditEmailTemplates` | ✓ | ✓ |  |  |  |  |
| `canBypassPayment` | ✓ | ✓ |  | ✓ |  |  |
| `canAccessBandRoom` | ✓ |  |  |  |  | ✓ |

`admin` short-circuits all permission checks (`useRolePermissions.ts`: `if (isAdmin) return true`).

### 6.3 Role-check pattern

- Postgres: `has_role(_user_id uuid, _role app_role)` and `has_any_role(_user_id uuid, _roles app_role[])` — both `STABLE SECURITY DEFINER` with `search_path=public`. Used inside RLS policies to avoid recursion. Role storage lives only in `public.user_roles` (never on `profiles`).
- React: `<ProtectedRoute permission="canX">` redirects to `/dashboard` on deny (or `/estimates` if fallback would loop).

### 6.4 Auth surfaces

| Mechanism | Tables / Functions | Notes |
|---|---|---|
| Email/password | Supabase Auth | Standard flow |
| PIN login | `user_login_pins`, `set_user_pin()`, `verify_user_pin()`, `verify_user_pin_with_lockout()`, `user_has_pin()` | 4–6 digit, `crypt+bf` hash, 5 failed attempts → 15-minute lockout. Edge fn: `pin-login` |
| MFA TOTP | Supabase Auth factors | `MFAEnrollment.tsx`, `MFAVerification.tsx`, `MFASettings.tsx` |
| MFA SMS | Edge fn `sms-mfa`, Twilio | `SMSMFAEnrollment.tsx` |
| Invitations | `invitations`, `handle_invitation_signup()` trigger, `get_invitation_by_token(p_token uuid)` RPC, edge fn `send-invite` | 1-hour acceptance window for role binding |
| Session timeout | `user_sessions`, `check_session_timeout(p_session_id, p_timeout_minutes default 480)`, `update_session_activity()` | 480-minute (8-hour) default |
| IP allowlist | Edge fn `verify-ip`, env `ALLOWED_OFFICE_IPS` | Non-admins restricted (`mem://auth/access-control-policy`) |
| Rate limiting | `api_rate_limits`, `check_rate_limit(p_identifier, p_endpoint, p_limit default 100, p_window_seconds default 60)` | Minute-bucketed |
| Audit | `audit_log` (237 rows total, 70 in last 30 days) | Insert+select policies |
| New-user bootstrap | `handle_new_user()` trigger → seeds `profiles` + `user_roles` (default `staff`); `set_admin_for_owner()` upgrades `mike@rolliworks.com` to `admin` | |

---

## 7. Data Model — Domain Groupings

> 118 tables grouped by domain. Column-level detail extracted from `information_schema.columns` (1460 columns total) is summarized; for full column lists query directly.

### 7.1 People & Identity (8 tables)
`customers`, `customer_addresses`, `customer_communication_permissions`,
`customer_shipping_bypass`, `profiles`, `user_roles`, `user_login_pins`, `user_sessions`.

- `customers` is the single source of truth for client identity (10,337 rows). Columns include normalized `email_normalized`, `phone_normalized` (maintained by trigger `trigger_update_customer_normalized` using `normalize_email()`, `normalize_phone()`), QBO link via `qbo_customer_id`, B2B hierarchy via `parent_customer_id` + `is_organization`, trade pricing flag `is_trade_pricing`, ship-direct flag `is_ship_direct`, optional `skip_shipping_info`.
- `profiles` mirrors `auth.users` (1:1 by `user_id`), seeded by `handle_new_user()` trigger.
- `user_roles` stores role assignments separately from `profiles` (privilege-escalation protection).

### 7.2 Watches, Estimates, Quotes (11 tables)
`watches`, `estimates`, `estimate_line_items`, `estimate_templates`,
`estimate_template_lines`, `estimate_attachments`, `model_references`,
`bracelet_models`, `reference_bracelets`, `calibers`, `caliber_sample_sheets`.

- `estimates.status` enum drives lifecycle: `draft → sent → (converted | declined | expired)`. Numbering via `get_next_estimate_number()` (5-digit, skips collisions, updates `settings.next_estimate_number`).
- `estimate_line_items` carries `service_subcategory_id`, `entered_part_number`, `part_id` (auto-relinked by trigger `auto_relink_estimate_line_item` calling `auto_relink_line_item()` — exact-match strategy only, no fuzzy).
- `model_references` (886 rows) — caliber/reference auto-sync source (`mem://features/intake/model-reference-auto-sync`).

### 7.3 Jobs, Tests, Custody (10 tables)
`jobs`, `job_status_history`, `job_activity_log`, `job_queue`, `timing_tests`,
`pressure_tests`, `inspections`, `inspection_requests`, `inspection_photos`,
`client_property`.

- Jobs: 1247 rows. All current rows have `status='intake'` per recent query — non-`intake` lifecycle handled by the companion app or `simple_job_status` workflow. Job IDs are year-prefixed (`generate_job_id()`: `YYNNNNN`) with a separate `get_next_job_id()` producing `E{n}` IDs.
- `client_property` (650 rows) — physical custody tracking with `custody_status` enum (`in_custody`, `released`) and `bin_id` placement.
- `job_status_history` populated by `log_job_status_change()` trigger on `jobs`.

### 7.4 Sales Orders & Fulfillment (14 tables)
`sales_orders`, `so_lines`, `shipments`, `shipment_logs`, `shipping_labels`,
`shipping_info_submissions`, `shipping_rates`, `pickup_sessions`,
`pickup_verification_codes`, `payment_bypass_log`, `vault_storage_fees`,
`pending_print_jobs`, `print_queue`, `label_prints`.

- `sales_orders.status` enum: `draft, open, partial_fulfilled, fulfilled, shipped, picked_up, cancelled`. Current breakdown: `shipped:641, fulfilled:102, draft:29, picked_up:27` (799 total).
- Numbering: `get_next_so_number()` → `SO-NNNNNN` from `settings.next_so_number`.
- `pickup_verification_codes` (561 rows): 8-char alphanumeric via `generate_pickup_code()`, looked up anonymously by `get_pickup_pass(p_code)` RPC.
- `shipping_labels` (482 rows): insurance amount, carrier metadata.

### 7.5 Inventory & Parts (20 tables)
`parts`, `part_aliases`, `part_categories`, `part_vendor_prices`, `vendor_parts`,
`vendor_price_history`, `vendors`, `caliber_parts`, `caliber_vendors`,
`inventory_stock`, `inventory_moves`, `inventory_adjustments`, `inventory_transfers`,
`transfer_lines`, `cycle_counts`, `cycle_count_lines`, `bins`, `locations`, `stores`,
`watchmaker_parts`.

- `parts` (5998 rows, 5832 active). `item_type` enum (8 values) discriminates inventory vs service vs client_property vs client_watch vs service_marker etc.
- Trigger `trigger_enforce_client_watch` calls `enforce_client_watch_item_type()` to auto-classify based on date-pattern in description.
- `inventory_stock` (1934 rows, total qty 11025) — per-location balances.
- Two parallel vendor-pricing tables exist: `vendor_parts` and `part_vendor_prices` (divergence noted in earlier dossier; both maintained).

### 7.6 Purchasing (9 tables)
`purchase_orders`, `po_lines`, `po_receipts`, `po_receipt_lines`,
`shop_work_orders`, `shop_work_order_lines`, `swo_lines`, `service_work_orders`,
`reorder_suggestions`.

- 80 POs total. Numbering: `get_next_po_number()` from `po_number_seq` → `PO-NNNNNN`. SWOs: `get_next_swo_number()` from `swo_number_seq` → `SWO-NNNNNN`.
- `reorder_suggestions` populated by `generate_reorder_suggestions()` — joins `parts` with `inventory_stock` (sum on hand) and preferred `vendor_parts` row; skips parts with existing pending/approved suggestions.

### 7.7 QuickBooks Online (6 tables)
`qbo_account_mappings`, `qbo_daily_journal`, `qbo_invoices`, `qbo_sync_log`,
`qbo_tokens`, `quickbooks_sync_log`.

- `qbo_invoices` is the largest table (38 MB). `qbo_tokens` stores OAuth tokens (432 kB).
- Sync orchestrated by edge functions `qbo-*` (see §9).

### 7.8 Reference & Reports (8 tables)
`appraisals`, `appraisal_templates`, `appraisal_field_options`,
`performance_verification_reports`, `performance_verification_grades`,
`report_presets`, `supplies`, `rw_hit_list_items`.

- `performance_verification_*` powers the Performance Verification Report (§4.9). Rating logic in `src/lib/performance-report-logic.ts`.
- `rw_hit_list_items` (3.3 MB, cached daily) — synced from RolliWorking via `hit-list` edge fn and `rw-status-cron`.

### 7.9 Messaging & Communications (4 tables)
`message_templates` (34 rows), `outbound_events`, `attachments`, `tc_acceptances`.

- `message_templates` is the single source for email/SMS template bodies. `mem://constraints/email-template-editability`: never hardcode; always pull from this table.
- `tc_acceptances` records terms-of-condition approvals from RolliWorking (`rw-tc-acceptance` edge fn, anonymous).

### 7.10 Public-Token / Anonymous Surfaces (10 tables)
`appointments`, `client_photo_requests`, `client_photo_submissions`,
`intake_leads`, `package_arrival_scans`, `package_scan_logs`,
`received_webhook_events`, `shipping_info_submissions`, `short_urls`,
`waitlist`.

- All accessed via anon-token RPCs or `verify_jwt = false` edge functions.
- `short_urls` (499 rows): `/go/:code` redirects, 6-char codes via `generate_short_code()`.

### 7.11 Infrastructure (8 tables)
`api_rate_limits`, `audit_log`, `cache_store`, `csv_import_staging`,
`camera_settings`, `scheduling_settings`, `scheduling_closed_dates`,
`scheduling_notes`, `scheduling_type_rules`, `settings`, `invitations`, `role_permissions`.

- `cache_store` is a generic key/jsonb cache with TTL (`cache_get`, `cache_set`, `cache_cleanup`).
- `scheduling_settings.max_appointments_per_slot` enforced by trigger `enforce_appointment_slot_capacity` calling `check_appointment_slot_capacity()`.

### 7.12 Misc / Other (6+ tables)
`bom_headers`, `bom_lines`, `line_items` (legacy), `service_categories`,
`service_subcategories`, `service_work_orders`.

---

## 8. Database Functions & Triggers

### 8.1 Function inventory (87 total in `public`; 31 user-defined, rest are `pg_trgm` operator funcs)

**Numbering generators**
- `get_next_estimate_number()` — 5-digit, skips existing
- `get_next_so_number()` — `SO-NNNNNN`
- `get_next_po_number()` — uses `po_number_seq`
- `get_next_swo_number()` — uses `swo_number_seq`
- `get_next_transfer_number()` — `TR-NNNNNN`
- `get_next_move_number()` — `MV-NNNNNN`
- `get_next_count_number()` — `CC-NNNNNN`
- `generate_job_id()` — `YYNNNNN`
- `get_next_job_id()` — `E{n}` style
- `generate_appraisal_number()` — `APR-YYYY-NNNN`
- `generate_pickup_code()` — 8-char alphanumeric
- `generate_short_code()` — 6-char URL-safe (alphabet excludes ambiguous chars)

**Domain functions**
- `get_job_profile(p_estimate_id uuid) → jsonb` — large composite payload aggregating estimate + customer + watch + line item flow classification + custody + sales order + appraisal. Used by downstream sync. See full body in earlier function dump.
- `get_pickup_pass(p_code text) → jsonb` — anon RPC for client pickup ticket lookup.
- `get_invitation_by_token(p_token uuid)` — anon RPC for invite acceptance.
- `get_photo_request_by_token(p_token uuid)` — anon RPC for client photo upload.
- `generate_reorder_suggestions() → int` — populates `reorder_suggestions` for parts at/below reorder point.
- `claim_next_job()` — queue claim from `job_queue`.
- `calculate_shipping_cost(p_item_type text, p_insured_value numeric)` — returns cheapest rate from `shipping_rates`.
- `search_parts(search_term text)` — unified search across `parts.part_number`, `vendor_parts.vendor_part_number`, `part_aliases.alias_sku`.
- `m3ke_get_pricing_facts(...)`, `m3ke_get_related_parts(...)` — analytical RPCs (8s statement timeout) for the M3KE pricing assistant.
- `get_band_room_weekly_totals()`, `get_reorder_candidates()` — report helpers.

**Auth / role / session**
- `has_role`, `has_any_role`, `is_authenticated`
- `set_user_pin`, `verify_user_pin`, `verify_user_pin_with_lockout`, `user_has_pin`
- `check_session_timeout`, `update_session_activity`
- `check_rate_limit`
- `handle_new_user`, `handle_invitation_signup`, `set_admin_for_owner`

**Cache / utility**
- `cache_get`, `cache_set`, `cache_cleanup`
- `normalize_email`, `normalize_phone`
- `update_updated_at_column` (generic), 12 table-specific `update_*_updated_at` variants
- `update_customer_normalized_fields`, `update_customer_addresses_updated_at`, `update_customer_communication_permissions_updated_at`
- `refresh_dashboard_stats` (refreshes `dashboard_stats` + `low_stock_parts` materialized views with `CONCURRENTLY`)

**Triggers (auto-classification & business rules)**
- `enforce_client_watch_item_type` (on `parts`) — auto-sets `item_type='client_watch'` when description contains a date pattern
- `auto_relink_line_item` (on `estimate_line_items`) — exact-match part relinking via `part_number`, `sku1`, `sku2`, `vendor_parts`, `part_aliases`, or description
- `auto_seed_inspection_request` (on `package_scan_logs`) — creates `inspection_requests` row on match
- `check_appointment_slot_capacity` (on `appointments`) — enforces `scheduling_settings.max_appointments_per_slot`
- `log_job_status_change` (on `jobs`)
- `update_customer_normalized_fields` (on `customers`) — keeps `email_normalized`, `phone_normalized`, `display_name` in sync

### 8.2 Trigger inventory (68)

```sql
SELECT event_object_table, trigger_name, event_manipulation, action_timing
FROM information_schema.triggers WHERE trigger_schema='public'
ORDER BY event_object_table, trigger_name;
```

Distributed across 56 tables. Pattern: most tables have a `BEFORE UPDATE` `update_<table>_updated_at` row-timestamp trigger; a few have `BEFORE INSERT/UPDATE` business-rule triggers as listed above. Full list omitted for brevity — re-run the SQL.

---

## 9. Edge Functions

74 functions under `supabase/functions/`. 44 are declared in `supabase/config.toml` with `verify_jwt = false` (public/webhook); the remainder require an authenticated JWT.

### 9.1 By category

**QuickBooks Online (16)** — auth, sync, push, pull, webhooks
```
qbo-accounts, qbo-cron-sync, qbo-customer-push, qbo-customer-sync,
qbo-estimate-sync, qbo-invoice-import, qbo-invoice-status, qbo-invoice-sync,
qbo-invoice-update, qbo-item-push, qbo-payment-email, qbo-payment-webhook,
qbo-po-receive, qbo-sales-order-push, qbo-so-pull
```
Plus `quickbooks_sync_log` / `qbo_sync_log` tables. Notes per `mem://integrations/qbo-synchronization-architecture`: empty SO rejection + 300ms race-condition lock.

**RolliWorking (watchmaker companion app) (8)**
```
rw-job-status, rw-parts-approved, rw-print-labels, rw-status-cron,
rw-tc-acceptance, rw-watchmaker-assignment, sync-model-to-rolliworking,
rollisuite-job-finished
```

**Shipping & fulfillment (9)**
```
request-shipping-label, return-shipping-info, scan-shipping-label,
send-fedex-pickup-notice, send-shipping-label-email, send-pickup-pass,
send-pickup-reminder, send-pickup-reminder-sms, shipping-bypass-optin
```

**Email / SMS notifications (Resend + Twilio) (12)**
```
send-appraisal-email, send-booking-link, send-bypass-optin, send-estimate-email,
send-invite, send-lead-followup, send-payment-reminder, send-payment-reminder-sms,
send-photo-request, send-po-email, send-receive-report, send-security-alert,
send-shipping-info-request, send-waitlist-email
```

**Public token / portal (8)**
```
book-appointment, client-tracking, rc-portal-invite, rc-timeline-status,
upload-client-photos, upload-photo, get-photo, shorten-url, wix-intake-webhook
```

**AI gateway (Lovable AI) (4)**
```
compare-signatures, docs-qa, extract-tolerance-sheet, polish-service-narrative
```
Prioritizes OpenAI vision models per `mem://architecture/lovable-ai-gateway`.

**Reference / model data (3)**
```
model-references-sync, pull-model-references, parts-pricing, parts-webhook
```

**Auth / security (4)**
```
decrypt-proxy-id, encrypt-proxy-id, pin-login, sms-mfa, verify-ip
```

**Cron / maintenance (5)**
```
gold-price-update, hit-list, maintenance-cron, vault-storage-cron, rw-status-cron
```

**Job lifecycle (1)**
```
job-status-update
```

### 9.2 `verify_jwt = false` (anon-callable, from `supabase/config.toml`)

44 functions, including all `qbo-*`, all webhooks (`parts-webhook`, `wix-intake-webhook`,
`qbo-payment-webhook`), and all client-token portals (`request-shipping-label`,
`return-shipping-info`, `upload-photo`, `upload-client-photos`, `get-photo`,
`scan-shipping-label`, `send-*-email`, `send-invite`, `send-waitlist-email`,
`rw-*`, `decrypt-proxy-id`, `encrypt-proxy-id`, `client-tracking`,
`rc-timeline-status`, `hit-list`, `docs-qa`, etc.). Each enforces its own token
or signature scheme inside the function body.

---

## 10. External Integrations

| Integration | Purpose | Surface | Secrets / config | Memory ref |
|---|---|---|---|---|
| **QuickBooks Online** | Customer/estimate/invoice/SO/Bill sync | 16 edge fns, `qbo_*` tables | OAuth tokens in `qbo_tokens` | `mem://integrations/qbo-synchronization-architecture` |
| **RolliWorking** | Watchmaker companion app — job status, parts approval, hit list, photos | 8 edge fns, `rw_hit_list_items`, `tc_acceptances` | Shared webhook secrets | `mem://integrations/rolliworking-*` |
| **Resend** | Transactional email | All `send-*-email` fns | `RESEND_API_KEY`; from `noreply@quotes.rolliworks.com` | `mem://auth/email-communications-identity` |
| **Twilio** | SMS notifications, MFA | `send-*-sms`, `sms-mfa`, `send-security-alert` | `TWILIO_*` env; E.164 normalization | `mem://features/security/security-alert-configuration` |
| **Cloudflare R2** | Inspection photo storage | `get-photo`, `upload-photo`, `upload-client-photos` | R2 access keys | `mem://integrations/rolliworking-photo-integration` |
| **Lovable AI Gateway** | OCR, signature compare, polish narrative, docs QA | `docs-qa`, `extract-tolerance-sheet`, `compare-signatures`, `polish-service-narrative` | `LOVABLE_API_KEY` | `mem://architecture/lovable-ai-gateway` |
| **FedEx** | Pickup notice | `send-fedex-pickup-notice` | Account-level | — |
| **Wix** | Intake form webhook (legacy, retirement planned per `docs/PORTAL_INVITE_SKETCH.md`) | `wix-intake-webhook` (anon) | Webhook signature | `mem://features/intake/intake-leads-workflow` |

---

## 11. Who Depends On RolliSuite

| Consumer | What it consumes | How | Notes |
|---|---|---|---|
| **RolliWorking** (watchmaker app) | Job status, customer info, parts approval, hit list, model references, photos | Outbound: `sync-model-to-rolliworking`, `rollisuite-job-finished`. Inbound: `rw-job-status`, `rw-parts-approved`, `rw-watchmaker-assignment`, `rw-tc-acceptance`. Cached locally in `rw_hit_list_items` | `mem://integrations/rolliworking-*` |
| **QuickBooks Online** | Customers, estimates, sales orders, invoices, POs/Bills | Bidirectional via `qbo-*` edge fns; OAuth tokens in `qbo_tokens` | RolliSuite is the system of record for line items; QBO is the system of record for accounting |
| **Client-facing portals** (`upload.rolliworks.com`) | Pickup tickets, shipping label requests, return info, photo uploads, appointment booking | Anon-token routes + RPCs (`get_pickup_pass`, `get_photo_request_by_token`, `get_invitation_by_token`) | Same React build, different domain |
| **Wix site** | Intake leads | `wix-intake-webhook` (anon) | Retirement planned per `docs/PORTAL_INVITE_SKETCH.md` |
| **Future RolliClient (RC) portal** | Client conversation history, invite tokens | `rc-portal-invite`, `rc-timeline-status` edge fns | Phase 2/3 — see sketch doc |
| **Lovable AI Gateway** | Model invocations (downstream of edge fns) | OpenAI/Gemini routed through gateway | — |
| **External recipients** | `[FILL IN — any downstream consumers not visible in source: BI tools, scheduled exports, third-party scanners, ERP, marketplaces, etc.]` |

---

## 12. Live Usage Data

All counts captured 2026-06-07 against the production database. Aggregates only — no PII.

### 12.1 Volumes (totals + time-windowed)

```sql
-- Run all metrics in one query for atomic snapshot
SELECT 'customers_total', count(*)::text FROM customers UNION ALL
SELECT 'estimates_total', count(*)::text FROM estimates UNION ALL
SELECT 'estimates_30d',  count(*)::text FROM estimates WHERE created_at >= now() - interval '30 days' UNION ALL
SELECT 'estimates_90d',  count(*)::text FROM estimates WHERE created_at >= now() - interval '90 days' UNION ALL
SELECT 'estimates_365d', count(*)::text FROM estimates WHERE created_at >= now() - interval '365 days' UNION ALL
SELECT 'sales_orders_total', count(*)::text FROM sales_orders UNION ALL
SELECT 'sales_orders_30d',   count(*)::text FROM sales_orders WHERE created_at >= now() - interval '30 days' UNION ALL
SELECT 'sales_orders_365d',  count(*)::text FROM sales_orders WHERE created_at >= now() - interval '365 days' UNION ALL
SELECT 'jobs_total', count(*)::text FROM jobs UNION ALL
SELECT 'parts_total', count(*)::text FROM parts UNION ALL
SELECT 'parts_active', count(*)::text FROM parts WHERE is_active=true UNION ALL
SELECT 'vendors_total', count(*)::text FROM vendors UNION ALL
SELECT 'purchase_orders_total', count(*)::text FROM purchase_orders UNION ALL
SELECT 'inspection_photos_total', count(*)::text FROM inspection_photos UNION ALL
SELECT 'client_property_total', count(*)::text FROM client_property UNION ALL
SELECT 'package_arrival_scans_total', count(*)::text FROM package_arrival_scans UNION ALL
SELECT 'package_arrival_scans_30d',   count(*)::text FROM package_arrival_scans WHERE created_at >= now() - interval '30 days' UNION ALL
SELECT 'pickup_verification_codes_total', count(*)::text FROM pickup_verification_codes UNION ALL
SELECT 'shipping_labels_total', count(*)::text FROM shipping_labels UNION ALL
SELECT 'short_urls_total', count(*)::text FROM short_urls UNION ALL
SELECT 'appraisals_total', count(*)::text FROM appraisals UNION ALL
SELECT 'message_templates_total', count(*)::text FROM message_templates UNION ALL
SELECT 'role_permissions_total', count(*)::text FROM role_permissions UNION ALL
SELECT 'audit_log_total', count(*)::text FROM audit_log UNION ALL
SELECT 'audit_log_30d',   count(*)::text FROM audit_log WHERE created_at >= now() - interval '30 days' UNION ALL
SELECT 'inspection_requests_total', count(*)::text FROM inspection_requests UNION ALL
SELECT 'so_lines_total', count(*)::text FROM so_lines UNION ALL
SELECT 'estimate_line_items_total', count(*)::text FROM estimate_line_items UNION ALL
SELECT 'invitations_total', count(*)::text FROM invitations UNION ALL
SELECT 'client_photo_requests_total', count(*)::text FROM client_photo_requests UNION ALL
SELECT 'client_photo_submissions_total', count(*)::text FROM client_photo_submissions UNION ALL
SELECT 'intake_leads_total', count(*)::text FROM intake_leads UNION ALL
SELECT 'intake_leads_30d',   count(*)::text FROM intake_leads WHERE created_at >= now() - interval '30 days' UNION ALL
SELECT 'model_references_total', count(*)::text FROM model_references UNION ALL
SELECT 'inventory_stock_rows', count(*)::text FROM inventory_stock UNION ALL
SELECT 'inventory_qty_total', COALESCE(sum(qty_on_hand),0)::text FROM inventory_stock UNION ALL
SELECT 'appointments_total', count(*)::text FROM appointments UNION ALL
SELECT 'appointments_future', count(*)::text FROM appointments WHERE appointment_date >= current_date UNION ALL
SELECT 'profiles_total', count(*)::text FROM profiles UNION ALL
SELECT 'user_roles_total', count(*)::text FROM user_roles UNION ALL
SELECT 'timing_tests_total', count(*)::text FROM timing_tests UNION ALL
SELECT 'pressure_tests_total', count(*)::text FROM pressure_tests;
```

| Metric | Value |
|---|---:|
| `customers_total` | 10,337 |
| `estimates_total` | 4,053 |
| `estimates_30d` | 308 |
| `estimates_90d` | 866 |
| `estimates_365d` | 2,586 |
| `sales_orders_total` | 799 |
| `sales_orders_30d` | 110 |
| `sales_orders_365d` | 799 |
| `jobs_total` | 1,247 |
| `parts_total` | 5,998 |
| `parts_active` | 5,832 |
| `vendors_total` | 820 |
| `purchase_orders_total` | 80 |
| `inspection_photos_total` | 8,583 |
| `client_property_total` | 650 |
| `package_arrival_scans_total` | 598 |
| `package_arrival_scans_30d` | 160 |
| `pickup_verification_codes_total` | 561 |
| `shipping_labels_total` | 482 |
| `short_urls_total` | 499 |
| `appraisals_total` | 26 |
| `message_templates_total` | 34 |
| `role_permissions_total` | 35 |
| `audit_log_total` | 237 |
| `audit_log_30d` | 70 |
| `inspection_requests_total` | 571 |
| `so_lines_total` | 6,143 |
| `estimate_line_items_total` | 23,085 |
| `invitations_total` | 4 |
| `client_photo_requests_total` | 207 |
| `client_photo_submissions_total` | 754 |
| `intake_leads_total` | 1,617 |
| `intake_leads_30d` | 369 |
| `model_references_total` | 886 |
| `inventory_stock_rows` | 1,934 |
| `inventory_qty_total` | 11,025 |
| `appointments_total` | 86 |
| `appointments_future` | 3 |
| `profiles_total` | 6 |
| `user_roles_total` | 6 |
| `timing_tests_total` | 0 |
| `pressure_tests_total` | 0 |

> Note: `timing_tests` and `pressure_tests` are zero — the operational test data
> presumably lives in the newer `performance_verification_*` tables instead.

### 12.2 Status breakdowns

```sql
SELECT status::text, count(*) FROM estimates GROUP BY status;
SELECT status::text, count(*) FROM sales_orders GROUP BY status;
SELECT status::text, count(*) FROM jobs GROUP BY status;
SELECT role::text,   count(*) FROM user_roles GROUP BY role;
```

| Estimates status | Count |  | Sales orders status | Count |
|---|---:|---|---|---:|
| `sent` | 1,812 |  | `shipped` | 641 |
| `declined` | 1,603 |  | `fulfilled` | 102 |
| `converted` | 533 |  | `draft` | 29 |
| `draft` | 105 |  | `picked_up` | 27 |
| **Total** | 4,053 |  | **Total** | 799 |

| Job status | Count |  | Role | Count |
|---|---:|---|---|---:|
| `intake` | 1,247 |  | `manager` | 2 |
|  |  |  | `staff` | 2 |
|  |  |  | `admin` | 1 |
|  |  |  | `front_desk` | 1 |

> Job statuses other than `intake` are not currently used — lifecycle progression
> appears to be tracked outside the `jobs.status` column (e.g. in `simple_job_status`
> rows, in the companion RolliWorking app, or via `sales_orders.status`).

### 12.3 Derived ratios

- Estimate → SO conversion rate (lifetime): `533 / 4053 = 13.2%`
- Estimate → SO conversion rate (last 365d): `799 / 2586 = 30.9%`
- Active parts ratio: `5832 / 5998 = 97.2%`
- Lead → estimate ratio (lifetime): `4053 / 1617 = 2.51 estimates per lead` (suggests multi-watch leads or direct estimate creation)
- Inspection photos per inspection: `8583 / 571 = 15.0` photos/inspection
- Pickup verification codes per SO: `561 / 799 = 70.2%` of SOs have a pickup code
- Photo requests fulfilled: `754 submissions / 207 requests = 3.64 photos per request`

### 12.4 Storage footprint

```sql
SELECT pg_size_pretty(sum(pg_total_relation_size(c.oid)))
FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
WHERE n.nspname='public' AND c.relkind='r';
```

Top 30 tables aggregate to ~85 MB; full schema dominated by `qbo_invoices` (38 MB) and
`estimate_line_items` (10 MB). See §5.3 for the full list.

### 12.5 Edge-function and analytics traffic

Edge-function invocation counts, Auth events, and Postgres log volumes are queryable via the Supabase analytics database (`function_edge_logs`, `auth_logs`, `postgres_logs`) and were not snapshotted into this dossier to keep it static. To pull on demand:

```sql
-- Edge function invocations, last 24h
SELECT m.function_id, count(*) as invocations
FROM function_edge_logs
CROSS JOIN unnest(metadata) AS m
WHERE timestamp >= now() - interval '24 hours'
GROUP BY m.function_id ORDER BY invocations DESC;
```

### 12.6 Cron jobs

`pg_cron` (v1.6.4) is installed. The `cron.job` table is not readable by the current
role — `[FILL IN — list of scheduled cron jobs as exposed by Supabase dashboard]`.
Known cron edge functions by naming convention: `qbo-cron-sync`, `gold-price-update`,
`hit-list`, `rw-status-cron`, `vault-storage-cron`, `maintenance-cron`.

---

## 13. Dependency Inventory

### 13.1 Frontend runtime (from `package.json`)

**Framework & routing**: `react@18.3.1`, `react-dom@18.3.1`, `react-router-dom@6.30.1`

**Server state**: `@tanstack/react-query@5.83.0`, `@supabase/supabase-js@2.90.1`

**UI primitives (Radix)**: 28 `@radix-ui/react-*` packages covering accordion, alert-dialog, aspect-ratio, avatar, checkbox, collapsible, context-menu, dialog, dropdown-menu, hover-card, label, menubar, navigation-menu, popover, progress, radio-group, scroll-area, select, separator, slider, slot, switch, tabs, toast, toggle, toggle-group, tooltip

**Forms / validation**: `react-hook-form@7.61`, `@hookform/resolvers@3.10`, `zod@3.25.76`

**Tables / dnd / charts**: `@dnd-kit/core@6.3`, `@dnd-kit/sortable@10.0`, `recharts@2.15.4`

**Dates / utilities**: `date-fns@3.6`, `date-fns-tz@3.2`, `clsx@2.1`, `tailwind-merge@2.6`, `class-variance-authority@0.7`

**Specialty / hardware**
- `pdfjs-dist@4.8.69` — PDF parsing
- `jspdf@2.5.2` — PDF generation
- `html2canvas@1.4.1` — Canvas rasterization
- `bwip-js@4.8` — Barcode rendering
- `jsbarcode@3.12.3`
- `qrcode.react@4.2`
- `html5-qrcode@2.3.8` — Browser camera QR scanning
- `jszip@3.10.1`
- `input-otp@1.4.2`
- `embla-carousel-react@8.6`
- `cmdk@1.1.1`
- `lucide-react@0.462`
- `sonner@1.7.4`, `vaul@0.9.9`, `next-themes@0.3`
- `react-day-picker@8.10`, `react-resizable-panels@2.1.9`

**Test**: `@playwright/test@1.58.2`

### 13.2 Build / dev

`vite@5.4.19`, `@vitejs/plugin-react-swc@3.11`, `typescript@5.8.3`,
`tailwindcss@3.4.17`, `@tailwindcss/typography@0.5.16`, `autoprefixer@10.4.21`,
`postcss@8.5.6`, `eslint@9.32`, `lovable-tagger@1.1.13`

### 13.3 Backend (Supabase managed)

Postgres extensions: see §5.6. Edge functions run on Deno managed by Supabase.

### 13.4 Companion monorepo (`app/`)

Per `app/apps/rollisuite-{api,web}/nixpacks.toml` and Railway configs, plus
`docs/RAILWAY_DEPLOYMENT.md`: Fastify API + Vite/React Web, Nixpacks builder,
Prisma migrations. Detailed dep list inside `app/apps/*/package.json` (not
inventoried here).

### 13.5 SaaS criticality (derived from integration surface)

| Service | Criticality | Failure impact |
|---|---|---|
| Supabase (Lovable Cloud) | Critical | App down |
| QuickBooks Online | High | Accounting sync halts; UI keeps working |
| Resend | High | Email comms halt |
| Twilio | Medium | SMS reminders + MFA SMS halt (TOTP still works) |
| Cloudflare R2 | High | Inspection photo viewing/uploading halts |
| Lovable AI Gateway | Medium | OCR, signature compare, narrative gen halt |
| RolliWorking | High | Bidirectional sync halts; jobs visible only in RS |
| FedEx API | Low | Manual pickup scheduling fallback |

### 13.6 Secrets reference (from `mem://auth/api-key-management`)

`RESEND_API_KEY`, `TWILIO_*`, `QBO_*`, `R2_*`, `LOVABLE_API_KEY`, `WIX_WEBHOOK_SECRET`, `ALLOWED_OFFICE_IPS`. All stored in Supabase function-level env vars or `supabase_vault`.

---

## 14. Stability Rules & Frozen Files

Verbatim from `STABILITY_RULES.md` (updated April 2026):

### Frozen Files (never modify without explicit bug report)
- `src/components/layout/AppLayout.tsx` — global nav, breaks everything if touched
- `src/contexts/AuthContext.tsx` — authentication state
- `src/integrations/supabase/client.ts` — database connection
- `src/integrations/supabase/types.ts` — generated DB schema, never hand-edit
- All `supabase/functions/qbo-*` — live QuickBooks sync
- All `supabase/functions/rw-*` — live RolliWorking integration

### High-Risk Files (touch only for confirmed bugs, surgical fixes only)
- `src/pages/estimates/EstimateDetailPage.tsx` (67 KB)
- `src/pages/inventory/PartsPage.tsx` (65 KB)
- `src/pages/estimates/CreateEstimatePage.tsx` (49 KB)
- `src/components/customers/CustomerSidePanel.tsx` (46 KB)
- `src/pages/sales/SalesOrderDetailPage.tsx` (34 KB)

### Safe to Touch (under 15 KB, lower regression risk)
- `src/pages/estimates/WaitlistPage.tsx`
- `src/pages/jobs/ShopTimePage.tsx`
- `src/pages/intake/IntakeLeadsPage.tsx`
- `src/pages/inventory/ReorderAlertsPage.tsx`
- `src/pages/inventory/CycleCountPage.tsx`

### Rules for Every Future Prompt
1. One problem per prompt — never combine fixes
2. State the exact file and exact symptom
3. Begin every prompt with "SURGICAL FIX ONLY"
4. Never ask for refactoring and bug fixes in the same prompt
5. Commit to GitHub before every session

### Do Not Build in Lovable (building in new system instead)
Inventory Transfers, Purchasing Receiving, Inventory Reports, Transaction Reports, Accounting, Stores & Locations, General Settings.

---

---

## 17. End-to-End Workflows (restored detail)

> Restored from the prior dossier pass — flow diagrams the regenerated version compressed out.

### 17.1 Lead → Cash (high level)
Lead → Estimate → (send) → Shipping label → Receive package → Inspect → Job → Sales Order →
Fulfillment (Pickup Station *or* Ship Station) → QBO invoice + client payment link.

### 17.2 Reorder cycle
Low stock detected → `generate_reorder_suggestions()` populates `reorder_suggestions` → PO
issued → receive against PO → stock updated.

### 17.3 Pickup verification
8-char code at `/pickup-pass/:code` → anon RPC `get_pickup_pass()` → "same item" confirmation
gate before release → `client_property.custody_status` flips to `released`.

### 17.4 Appointment booking
Public `/book` → DB capacity trigger validates slot → `appointments` row.

### 17.5 Inspection capture
```
Camera permission (FIRST async op — user gesture) → MediaStream
        ↓
Webcam → optional microscope camera handoff
        ↓
canvas → JPEG → upload-photo edge fn → R2 (path: YYYY-MM-DD/REFERENCE/...)
        ↓
inspection_photos row
```

### 17.6 Repair flow classification
Markers in `parts.item_type='service_marker'` map to flows (computed inside `get_job_profile()`
via service-code presence `W-`, `B-`, `P-`, `PM-`):

| Marker | Flow | Departments |
|---|---|---|
| `[W]` | STANDARD_FLOW | watchmaker |
| `[B]` | BAND_ONLY_FLOW | band |
| `[P]` | BAND_ONLY_FLOW | polish |
| `[PM]` | BAND_ONLY_FLOW | precious_metals |
| `[WB]` | SPLIT_FLOW | watchmaker + band |
| `[WP]` | SPLIT_FLOW | watchmaker + polish |
| `[WBP]` | SPLIT_FLOW | watchmaker + band + polish |

---

## 18. Critical Constraints — Core Rules (verbatim, restored)

> Hard-won operational rules from `mem://`. Restored from the prior pass. These are constraints
> any rebuild must honor.

- **Email**: never hardcode; use `message_templates`.
- **Hardware**: USB camera init must be the FIRST async operation (preserves user gesture).
- **Supabase limits**: client-side batching in chunks of 500; two-step queries to bypass
  OR/URL-length limits.
- **Printing**: Blob URLs + `win.print()` from parent window to bypass popup blockers.
- **Data integrity**: detail pages use `orderInitialized` ref so DB refetches don't overwrite
  user form state.
- **Routing**: client routes on `upload.rolliworks.com`; internal staff routes on primary
  domain.
- **Stability**: SURGICAL FIXES ONLY. Frozen Files: `AppLayout`, `AuthContext`,
  `supabase/*`, `qbo-*`, `rw-*`.
- **Deployment**: Railway monorepo (Fastify API + Vite/React Web) via Nixpacks + Prisma
  migrations.
- **Auth**: non-admins restricted by `ALLOWED_OFFICE_IPS`.

### 18.1 Memory cross-reference (selected — full list in `mem://index.md`, 90+ entries)

| Memory ID | Governs |
|---|---|
| `mem://features/purchasing/purchase-order-workflow` | Purchasing |
| `mem://features/intake/dynamic-received-items-extraction` | Intake |
| `mem://features/shipping/shipping-label-requests-management` | Ship Station |
| `mem://integrations/rolliworking-parts-approval` | RW integration |
| `mem://constraints/email-template-editability` | Email |
| `mem://auth/email-communications-identity` | Email |
| `mem://integrations/rolliworking-repair-workflow` | RW integration |
| `mem://features/search/global-search-capabilities` | Global search |
| `mem://features/security/security-alert-configuration` | Twilio/SMS alerts |
| `mem://features/security/signature-verification-system` | Pickup/signature |
| `mem://features/sales/appointment-scheduling` | Public booking |
| `mem://integrations/qbo-synchronization-architecture` | QBO |
| `mem://features/inspection/scantron-blueprint` | Inspection / AI OCR |
| `mem://architecture/lovable-ai-gateway` | AI gateway |
| `mem://architecture/realtime-state-synchronization` | Realtime |
| `mem://constraints/browser-hardware-access` | Inspection camera |
| `mem://features/customers/organization-hierarchy-mapping` | Customers/B2B |
| `mem://features/pickup/proxy-id-security` | Pickup security |
| `mem://features/sales/daily-hit-list` | Daily Hit List |
| `mem://features/sales/reminder-system` | Reminders |
| `mem://architecture/client-facing-domain-policy` | Domain split |
| `mem://features/inventory/cycle-count-session-management` | Inventory |
| `mem://features/pickup/pickup-station-routing-logic` | Pickup routing |
| `mem://features/sales/payment-bypass-audit-system` | Payment bypass audit |
| `mem://features/reports/dashboard-metrics` | Reports |
| `mem://features/reports/watchmaker-productivity-tracking` | Reports (Revenue/$550 per watch) |
| `mem://features/reports/department-revenue` | Reports |
| `mem://features/repair/turnaround-targets` | Turnaround (A=27w, V=12w, M/P=4w, B=5–8w) |
| `mem://features/sales/vault-storage-fee-automation` | Vault fee (auto $50) |
| `mem://features/labels/asset-label-spec` | Labels |
| `mem://infrastructure/print-window-architecture` | Printing |
| `mem://constraints/supabase-query-optimization` | Global query patterns |
| `mem://auth/access-control-policy` | Office IP allowlist |
| `mem://auth/api-key-management` | Secrets |


## 15. Glossary

| Term | Definition |
|---|---|
| **SO** | Sales Order. Table `sales_orders`. Prefix `SO-NNNNNN`. |
| **PO** | Purchase Order. Table `purchase_orders`. Prefix `PO-NNNNNN`, sequence `po_number_seq`. |
| **SWO** | Shop Work Order — internal repair labor record. Table `shop_work_orders`. Prefix `SWO-NNNNNN`. |
| **Estimate** | Pre-sale quote. Table `estimates`. 5-digit ID via `get_next_estimate_number()`. |
| **Job** | Active work record. Table `jobs`. ID `YYNNNNN` via `generate_job_id()`. |
| **Custody** | Physical possession of a client's watch. Tracked via `client_property.custody_status` (`in_custody` / `released`) and `bin_id`. |
| **Scantron** | Internal name for the handwritten polish-grade extraction workflow. See `mem://features/inspection/scantron-blueprint`. |
| **Hit List / Daily Hit List** | Prioritized daily task queue, synced from RolliWorking. See `mem://features/sales/daily-hit-list`. |
| **Band Room** | Bracelet/strap department (department code `B`). Restricted role: `band_room`. |
| **Kiosk** | Public in-store self-check-in surface at `/kiosk`. Offline-capable multi-select intake. |
| **Department codes** | `W` = Watchmaker, `B` = Band, `P` = Polish, `PM` = Precious Metals. Used as service-code prefixes (`W-`, `B-`, `P-`, `PM-`). |
| **Service marker** | A `parts.item_type='service_marker'` row used as a flow-classification token in estimate line items: `[W]`, `[B]`, `[P]`, `[PM]`, `[WB]`, `[WP]`, `[WBP]`. Parsed by `get_job_profile()`. |
| **Polish Grade** | 1–10 scale captured during inspection. See `mem://features/inspection/scantron-blueprint`. |
| **Turnaround targets** | Per `mem://features/repair/turnaround-targets`: `A=27w` (full restoration), `V=12w` (vintage), `M/P=4w` (modern/polish), `B=5–8w` (band). |
| **Breakeven target** | Dashboard metric, formula in `mem://features/reports/dashboard-metrics`. |
| **Proxy ID** | AES-256-GCM encrypted client identifier for pickup verification. 24-month purge. See `mem://features/pickup/proxy-id-security`. |
| **CW-prefix part** | A part_number of the form `CW-<3-char-brand>-<8-char-serial-prefix>` representing a client-owned watch (`item_type='client_watch'`). See `docs/SO_TO_PERFORMANCE_REPORT_PREFILL.md`. |
| **IFS** | Internal label format generated at `/intake/ifs-label-generator`. See `mem://features/intake/ifs-label-generator`. |
| **RW / RolliWorking** | The watchmaker companion app. Bidirectional sync via `rw-*` and `sync-model-to-rolliworking` edge fns. |
| **RC / RolliClient** | Planned client-conversation portal (Phase 2/3). See `docs/PORTAL_INVITE_SKETCH.md`. |
| **Pickup Pass** | Anonymous 8-char ticket at `/pickup-pass?code=XXXXXXXX` resolved via `get_pickup_pass(p_code)` RPC. |

---

## 16. Open Questions — `[FILL IN]` Index

Items in this dossier that require business input. Sweep this list and replace each placeholder.

1. **§2** Legal entity / company name — `[FILL IN — legal entity that operates RolliSuite]`
2. **§2** Physical location(s) / shop address(es) — `[FILL IN — primary shop address and any additional locations]`
3. **§2** Business description — `[FILL IN — what the business does, in plain language]`
4. **§2** Primary customer segments — `[FILL IN — e.g. retail watch repair clients, B2B trade accounts, wholesale, etc.]`
5. **§2** Strategic goals for next 6–12 months — `[FILL IN — north-star goals the software should support]`
6. **§2** Team size and seat count per role — `[FILL IN — headcount per role: admin / manager / office / front_desk / staff / band_room]`
7. **§3.5** Production state of companion monorepo — `[FILL IN — current production status of the Railway monorepo: live / staging-only / not yet deployed]`
8. **§11** External RolliSuite consumers not visible in source — `[FILL IN — any downstream consumers not visible in source: BI tools, scheduled exports, third-party scanners, ERP, marketplaces, etc.]`
9. **§12.6** Cron job inventory — `[FILL IN — list of scheduled cron jobs as exposed by Supabase dashboard]`

> When updating, keep the literal `[FILL IN — …]` token shape so a downstream parser
> (or another Lovable session) can sweep for remaining placeholders with a single
> regex: `\[FILL IN — [^\]]+\]`.

---

*End of dossier. Regenerate by re-running the SQL blocks in §5 and §12 against the
production database, then re-deriving §3–§11 from `src/` and `supabase/`.*
