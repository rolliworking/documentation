# RolliWorking (RW) — Context Dossier

> **Merged canonical version (2026-06-07).** Technical body derived by Lovable from the live RW
> codebase + read-only SELECT queries (project ref `pkgnrcfqrldwjibghefm`). Section 0 below is
> the cross-verified findings layer and is AUTHORITATIVE — it overrides any conflicting
> inference in the technical body. Re-apply Section 0 on top if the body is ever regenerated.
>
> **Companion docs:** `NORTH-STAR.md`, `RS_CONTEXT_DOSSIER.md`, `RC_CONTEXT_DOSSIER.md`.

---

## 0. Verified Findings & Cross-App Confirmations (read first — authoritative)

### 0.1 Architecture: THREE separate databases (corrects prior assumption)
- **RW has its OWN Supabase project** (`pkgnrcfqrldwjibghefm`), separate from RS
  (`djbjwcoddddywkgljuja`) and RC (`ibwrjsmuvrqtokoqpogu`). **All three apps have their own
  databases.** RW reaches RS only via specific edge functions (`rollisuite-status-push`,
  `hit-list-push`, and an `ai-assistant` cross-project read for photos) — never a shared schema.
- **IMPORTANT for the rebuild:** the goal of a "shared Supabase backend" for RS+RW is **NOT the
  current reality.** Today it is three databases talking through contracts/webhooks.
  INTEGRATIONS.md's "RS+RW share one Supabase" is aspirational or stale. The rebuild's
  shared-backend ambition is a real migration, not a description of what exists.

### 0.2 The job lifecycle lives in RW — full state machine now documented
This is the workflow RS could not show (RS's job log only had `webhook_intake` because RS hands
off immediately). The real pipeline, server-enforced by the `set_job_status` RPC:

`intake → inspection → waiting_approval → in_queue → uncased → in_progress → parts_approval →
parts_on_order → in_testing → waiting_components → finished`

Live distribution (526 jobs): finished 330, in_progress 75, waiting_approval 65, in_queue 31,
in_testing 20, parts_approval 5. (`intake`, `uncased`, `parts_on_order`, `waiting_components`
are valid but transient — 0 live rows; they appear in `job_status_changes`, so the pipeline
does traverse them.) A hard server guard blocks `finished` until all `job_components` are
reunited. Full transition table with triggers/authors/side-effects in §4.

### 0.3 This explains the RC portal-flow problem end-to-end (cross-app)
The "confusing client portal states" (RC priority) trace to here: **RW sets states like
`uncased` and `in_testing`** → pushes them to RS via `rollisuite-status-push` → RS serves them
to RC's portal → RC renders the raw label to the customer (RC has no hardcoded flow; it displays
whatever RS sends — see RC dossier §0.2/§5). So the fix for client-facing status wording is a
question of **what vocabulary RW emits and how RS relays it**, not RC display logic. The full
chain is now documented across all three dossiers.

### 0.4 The RT "downgrade" seam is NOT wired (gap vs. spec)
INTEGRATIONS.md §3.1 specifies RolliTime pushing "in testing" to RW and triggering a
"downgraded" return-to-watchmaker. **Today this has ZERO implementation in RW** — no
`rollitime-*` edge function, no inbound RT endpoint, no `process_level`/`downgraded_at` column.
RW *has* an `in_testing` status (20 live jobs) reachable when a watchmaker scans the testing
label, but the RT→RW automated downgrade path does not exist. Currently a watchmaker would
manually move the job back to `in_progress`, with no audit trail distinguishing an RT-driven
downgrade from a manual one. This is a real spec-vs-reality gap for the RT integration work.
- `[FILL IN — Michael: is RT deployed today or planned? If deployed, its base URL/project ref
  and the contract shape it will POST to RW.]`
- `[FILL IN — Michael: desired RW endpoint + payload for RT results (e.g. `rollitime-result`
  POSTing estimate_number, result, failure_reason?, tested_at).]`

---

## 1. System overview

**RolliWorking (RW)** is the watchmaker-shop operations app sitting downstream
of **RolliSuite (RS)**, the front-of-house estimating/POS system. RS originates
the job (intake, estimate, customer relationship) and pushes it into RW via
webhook. RW then owns the full repair lifecycle: inspection, client approval,
parts ordering, watchmaker assignment, bench work, sub-component routing
(band repair, polish, precious metals, refinishing), testing, and finished/
ready-to-ship hand-off back to RS.

**Stack.**

| Layer | Implementation |
| --- | --- |
| Frontend | React 18 + Vite 5 + TypeScript 5 + Tailwind v3 + shadcn/ui |
| Data layer | `@tanstack/react-query` over `@supabase/supabase-js` |
| Backend | Supabase (Postgres + RLS + Edge Functions, Deno runtime) |
| Auth | Supabase Auth (email + Google), TOTP MFA + 4-digit PIN, 8-hour session timeout |
| Realtime | Supabase Realtime not in active use; React Query polling + invalidation |
| Hosting | Lovable preview + custom domains: `app.rolliworks.com`, `rolliworking.com`, `www.rolliworking.com` (per `project_urls`) |
| External storage | RS-side Cloudflare R2 for inspection photos (proxied via RS `get-photo` edge function); RW-side Supabase Storage buckets `scantron-templates` (public) and `scanner-uploads` (private) |

**Users (derived from `user_roles.role` enum and `permissions` map).**

```sql
-- SELECT enum values
SELECT unnest(enum_range(NULL::app_role));
```

Roles: `owner`, `manager`, `staff`. The frontend `useUserRole` hook
(`src/hooks/use-user-role.ts`) gates: owner/manager can edit jobs and set part
prices; staff routes default to `/parts-history` instead of `/work-queue`
(`src/config/routes.tsx` `HomeRedirect`).

| Persona | Primary surface | Notes |
| --- | --- | --- |
| Watchmaker-room supervisor / shop owner | `/work-queue`, `/daily-hit-list`, `/shop-floor`, `/analytics` | Assigns watchmakers, approves status overrides |
| Watchmaker (bench technician) | `/shop-floor`, `/station-scanner`, `/component-status`, `/watchmaker-activity` | Scans labels, updates component status |
| Band / polish / precious-metals techs | `/band-work-queue`, `/follow-up-queue`, `/component-lookup` | Operate `job_components` with `department` ∈ {band_repair, refinishing, precious_metals} |
| Front-of-house / customer service | `/clients`, `/history`, `/client-replies`, `/email-templates`, `/parts-history` | Tracks inspection + parts approval responses |
| Administrator (owner) | `/users`, `/audit-logs`, `/data-management`, `/wiki`, `/settings/*` | Role + permission, scantron templates, backfill tools |

**[FILL IN — legal/business identity behind RolliWorks (entity name, location,
ownership relationship to the RS business)]**

**[FILL IN — headcount of bench watchmakers vs. band/polish techs vs. front
office expected to use RW day-to-day]**

---

## 2. Database architecture — RW vs RS

**RW has its OWN Supabase database. RW and RS do NOT share a database.**

Evidence:

| Project | Supabase project ref | Edge-function base URL |
| --- | --- | --- |
| **RW (this project)** | `pkgnrcfqrldwjibghefm` | `https://pkgnrcfqrldwjibghefm.supabase.co` |
| **RS (separate project)** | `djbjwcoddddywkgljuja` | `https://djbjwcoddddywkgljuja.supabase.co` |

The RS project ref appears as a hard-coded outbound target in three RW edge
functions, never as a database URL the RW client connects to:

- `supabase/functions/rollisuite-status-push/index.ts` →
  `https://djbjwcoddddywkgljuja.supabase.co/functions/v1/job-status-update`
- `supabase/functions/hit-list-push/index.ts` →
  `https://djbjwcoddddywkgljuja.supabase.co/functions/v1/hit-list`
- `supabase/functions/ai-assistant/index.ts` builds a second
  `createClient(RS_URL, RS_KEY)` to read RS tables (`estimates`,
  `client_property`, `inspections`, `inspection_photos`) for photo lookup
  — this is a cross-project read using an RS-issued service key, not a shared
  schema.

```sql
-- Confirm the RW DB identity
SELECT current_database(), current_user;
-- → postgres, sandbox_exec
```

```sql
-- Confirm there is no rolllisuite/rs schema inside RW
SELECT schema_name FROM information_schema.schemata
WHERE schema_name NOT IN ('pg_catalog','information_schema','pg_toast');
-- public + supabase-managed schemas only
```

Every table listed in §3 is RW-owned. RS owns its own copies of `customers`,
`estimates`, `inspections`, `inspection_photos`, `client_property` etc. on the
RS project; RW does not access them by SQL except through the ai-assistant
cross-project read path noted above.

---

## 3. Full schema

Tables flagged **★ workflow-core** are the spine of the repair lifecycle in §4.

| Table | Workflow? | Columns (type) |
| --- | --- | --- |
| `jobs` | ★ | `id:uuid, inspection_id:uuid, status:text, service_type:text, due_date:date, work_started:bool, work_started_at:timestamptz, outsourced_tasks:jsonb, last_update_email_sent:timestamptz, created_at:timestamptz, updated_at:timestamptz, services:jsonb, custom_tasks:jsonb, repair_tasks:jsonb, is_movement_service:bool, parts_approval_needed:bool, parts_details:text, estimated_cost:numeric, notes:text, client_id:uuid, client_name:text, client_email:text, watch_brand:text, watch_model:text, serial_number:text, intake_date:date, needs_liability_waiver:bool, waiver_reason:text, waiver_signed:bool, sent_email_templates:jsonb, estimate_number:text, parts_requests:jsonb, parts_approval_status:text, in_testing_at:timestamptz, assigned_watchmaker:text, used_parts:jsonb, finished_date:date, uncased_at:timestamptz, received_bracelet:bool, received_case:bool, flow:text, dept_w:bool, dept_b:bool, dept_p:bool, dept_pm:bool, bracelet_description:text, received_components:text[]` |
| `inspections` | ★ | `id, watch_id, inspection_type, status, dial_condition:jsonb, hands_condition:jsonb, bezel_condition:jsonb, crown_condition:jsonb, case_condition:jsonb, crystal_condition:jsonb, bracelet_condition:jsonb, total_estimate:numeric, pricing_details:jsonb, waiver_required, waiver_signed, notes, created_by, created_at, updated_at, job_type, inspection_number, use_html_email, job_types:text[], department_tag` |
| `inspection_approvals` | ★ | `id, inspection_id, client_name, client_email, approval_items:jsonb, tc_agreed, approved_by_name, status, approved_at, created_at, updated_at, tc_ip_address, tc_user_agent, tc_version_url, group_id, polish_answers:jsonb, question_answers:jsonb, client_notes, confirmation_sent_at` |
| `parts_approvals` | ★ | `id, job_id, parts_items:jsonb, status, approved_by_name, approved_at, client_name, client_email, client_notes, tc_agreed, tc_ip_address, tc_user_agent, created_at, updated_at` |
| `job_components` | ★ | `id, job_id, component_type, label_id, status, assigned_to, department, status_changed_at, created_at, notes, cancelled_reason, cancelled_by, cancelled_at, assigned_watchmaker_id, assigned_dept_tech_id, assignee_name, assignee_initials, station_id` |
| `job_component_logs` | ★ | `id, component_id, action, old_value, new_value, changed_by, created_at, job_id, from_status, to_status, assigned_to, department, notes, is_change_order, is_override, station_id` |
| `job_component_notes` |   | `id, job_component_id, note, created_by, created_at` |
| `job_sub_components` | ★ | `id, job_id, watch_head_component_id, sub_component_type_id, status, assigned_to, condition_notes, created_at, updated_at` |
| `job_sub_component_logs` | ★ | `id, sub_component_id, job_id, from_status, to_status, assigned_to, changed_by, changed_at, notes` |
| `sub_component_types` |   | `id, name, can_outsource, is_active, sort_order, created_at` |
| `job_status_changes` | ★ | `id, job_id, previous_status, new_status, job_type, changed_at, changed_by` |
| `watches` | ★ | `id, customer_id, brand, model, reference_number, estimate_number, target_date, created_at, updated_at` |
| `customers` |   | `id, name, email, created_at, updated_at, created_by, phone` |
| `outsource_dispatches` | ★ | `id, job_id, vendor_id, vendor_name_override, sent_by, sent_at, tracking_number, expected_return, condition_notes, returned_at, received_by, return_condition_notes, created_at` |
| `outsource_dispatch_items` | ★ | `id, dispatch_id, sub_component_id, condition_notes` |
| `vendors` |   | `id, name, vendor_type, contact_name, phone, email, address, notes, is_active, created_at, updated_at` |
| `possession_scan_sessions` | ★ | `id, conducted_by, technician_id, started_at, completed_at, items_expected:jsonb, items_scanned:jsonb, missing_items:jsonb, unexpected_items:jsonb, notes, status` |
| `liability_waivers` |   | `id, job_id, status, customer_name, customer_email, watch_brand, watch_model, serial_number, estimate_number, reference_number, dial_defects, hand_defects, additional_components, additional_info, waiver_date, completed_at, created_at, updated_at, waiver_number` |
| `process_stages` |   | `id, department, value, label, is_default, is_active, sort_order, created_at` |
| `service_codes` |   | `id, code, job_type, weeks, label, is_bracelet, is_movement_service, is_active, sort_order, created_at, updated_at` |
| `department_technicians` |   | `id, name, initials, department, is_active, sort_order, created_at, updated_at, tech_code` |
| `watchmakers` |   | `id, name, initials, is_active, created_at, updated_at, weekly_target, weekly_testing_target, tech_code` |
| `inspection_questions` |   | `id, key, label, description, is_active, created_at, updated_at, show_on_client, required_for_submission, render_as_scale` |
| `inspection_rules` |   | `id, section, job_types:text[], action, target_question, description, is_active, sort_order, created_at, updated_at` |
| `custom_notes` |   | `id, section, note, created_by, created_at` |
| `note_usage` |   | `id, section, note, usage_count, last_used_at, created_at` |
| `email_templates` |   | `id, name, type, subject, body, is_active, created_at, updated_at` |
| `model_references` |   | `id, part_number, brand, model, caliber, notes, source, created_at, updated_at` |
| `approval_read_status` |   | `id, approval_id, user_id, is_read, is_archived, read_at, archived_at, created_at, updated_at` |
| `wiki_articles` |   | `id, title, content, category, sort_order, created_by, updated_by, created_at, updated_at` |
| `audit_logs` |   | `id, user_id, user_email, action, resource_type, resource_id, details:jsonb, ip_address, user_agent, created_at` |
| `login_sessions` |   | `id, user_id, ip_address, user_agent, device_fingerprint, location_info:jsonb, is_current, last_active_at, created_at` |
| `scanner_uploads` |   | `id, storage_path, filename, status, processed_at, created_at` |
| `scanner_settings` |   | `id, user_id, folder_path, poll_interval_seconds, auto_process, created_at, updated_at` |
| `scantron_templates` |   | `id, version, label, storage_path, is_active, uploaded_by, created_at` |
| `profiles` |   | `id, user_id, full_name, email, role, created_at, updated_at` |
| `user_roles` |   | `id, user_id, created_at, updated_at, role(app_role)` |
| `user_security` |   | `id, user_id, pin_hash, totp_secret, totp_enabled, totp_verified, session_timeout_minutes, login_notification_enabled, created_at, updated_at` |
| `permissions` |   | `id, key, name, description, category, created_at` |
| `role_permissions` |   | `id, role(app_role), permission_key, created_at` |

**Foreign keys.** `information_schema.table_constraints` reports **zero**
declared FKs in the `public` schema:

```sql
SELECT count(*) FROM information_schema.table_constraints
WHERE constraint_type='FOREIGN KEY' AND table_schema='public';
-- → 0
```

All relationships are by-convention only (uuid columns named
`<table>_id`). The implicit relationship map:

```
customers.id ← watches.customer_id
watches.id   ← inspections.watch_id
inspections.id ← jobs.inspection_id
                 inspection_approvals.inspection_id
jobs.id      ← parts_approvals.job_id
                 job_components.job_id
                 job_sub_components.job_id
                 outsource_dispatches.job_id
                 liability_waivers.job_id
                 job_status_changes.job_id
                 job_component_logs.job_id
                 job_sub_component_logs.job_id
job_components.id            ← job_component_notes.job_component_id
                               job_component_logs.component_id
                               job_sub_components.watch_head_component_id
job_sub_components.id        ← job_sub_component_logs.sub_component_id
                               outsource_dispatch_items.sub_component_id
sub_component_types.id       ← job_sub_components.sub_component_type_id
vendors.id                   ← outsource_dispatches.vendor_id
outsource_dispatches.id      ← outsource_dispatch_items.dispatch_id
auth.users.id                ← user_roles.user_id, profiles.user_id, user_security.user_id, login_sessions.user_id, audit_logs.user_id, custom_notes.created_by, customers.created_by, inspections.created_by, watchmakers (via tech_code) etc.
```

---

## 4. The job lifecycle (state machine)

### 4.1 `jobs.status` — the main pipeline

Observed in `job_status_changes.new_status` (history) and `jobs.status` (live):

```sql
SELECT DISTINCT new_status FROM public.job_status_changes ORDER BY 1;
-- finished, in_progress, in_queue, in_testing, inspection,
-- parts_approval, parts_on_order, uncased, waiting_approval
```

Allowed set is enforced server-side by the `public.set_job_status(job_id, new_status)`
RPC (`SECURITY DEFINER`), which whitelists:

```
intake, inspection, waiting_approval, in_queue, uncased, in_progress,
parts_approval, parts_on_order, in_testing, waiting_components, finished
```

Transition triggers and authors:

| From → To | Trigger | Author | Side-effects |
| --- | --- | --- | --- |
| (n/a) → `intake` | RS pushes `WORK_COMPLETED` webhook → `rollisuite-webhook` creates/updates the row, OR a manager creates a job manually via `/new-job` | RS service / manager | `inspection_id` linked when an inspection already exists for the watch |
| `intake` → `inspection` | Watchmaker-room opens `/inspections/new` for the watch | Manager / owner (RLS) | Writes `inspections` row, `dial_condition`/`hands_condition`/... jsonb |
| `inspection` → `waiting_approval` | `send-inspection-email` invoked from inspection page; creates an `inspection_approvals` row with `status='pending'` and emails the customer | Manager / owner | `inspections.status='sent'`; if `inspections.job_types` contains polish/PM, `polish_answers` are required on submission |
| `waiting_approval` → `in_queue` | Customer hits approve on the public approval page → `submit-inspection-approval` flips `inspection_approvals.status='approved'` and POSTs to the RS `submit-inspection-approval` endpoint | Customer (anon) | `parts_approval_needed` and `dept_*` flags are written by the inspection step; the watch enters the bench queue |
| `in_queue` → `uncased` | Watchmaker scans the uncasing label at `/station-scanner` | Watchmaker | `jobs.uncased_at=now()`; status push to RS |
| `uncased` → `in_progress` | Watchmaker starts bench work (scan or manual) | Watchmaker | `work_started=true, work_started_at=now()` |
| `in_progress` → `parts_approval` | Watchmaker adds a parts request via `/parts-request` and submits for client approval. RPC `add_parts_request` writes `parts_requests` jsonb + `parts_approval_status='pending'`. An entry is created in `parts_approvals` | Manager / watchmaker | Public link emailed; tracked in `/parts-history` |
| `parts_approval` → `parts_on_order` | Customer approves → `submit-parts-approval` → fires `rollisuite-parts-approved` to RS for SO creation; sets `parts_approvals.status='approved'` | Customer (anon) | RS allocates stock; `rollisuite-part-allocation` records allocation back |
| `parts_on_order` → `in_progress` | RS confirms stock fulfilled (`rollisuite-so-fulfilled` webhook) | RS service | Status restored to in-progress |
| `in_progress` → `in_testing` | Watchmaker scans the testing label | Watchmaker | `in_testing_at=now()`; status push to RS |
| `in_testing` → `in_progress` | "Downgraded" return-to-watchmaker (see §6) | RolliTime (RT) — see §6 | Currently **NOT WIRED** — see §6 |
| `in_testing` → `waiting_components` | All components on the watch reach `reunited`/`waiting_components`/`fulfilled`; the job is awaiting band/polish/PM return before final assembly | Watchmaker | Guarded by `set_job_status` |
| `waiting_components` → `finished` | All non-cancelled `job_components.status` ∈ {`waiting_components`,`reunited`} (guard in `set_job_status`); manager/watchmaker hits "Finished" | Manager / watchmaker | `finished_date=current_date`; RS receives `ready_to_ship` via `rollisuite-status-push`; `rollisuite-job-finished` may fire |
| any → `finished` (block) | If any non-cancelled `job_components` is not in `waiting_components`/`reunited` the RPC raises `Cannot finish: not all components are reunited` | — | Hard server-side guard |

### 4.2 Watchmaker assignment

- Single primary watchmaker on the job: `jobs.assigned_watchmaker` (text — the
  initials, e.g. `EG`, `CS`, `WG`).
- Set via `assign_watchmaker(_job_id, _assigned_watchmaker)` RPC, which also
  fires the `rollisuite-watchmaker-assign` edge function (fire-and-forget)
  pushing `{ estimate_number, assigned_watchmaker, assigned_at, status }` to
  RS so RS reflects the same assignment.
- Component-level assignments live on `job_components.assigned_watchmaker_id`
  / `assigned_dept_tech_id` / `assignee_initials` and are emitted in
  `job_component_logs`.

### 4.3 Parts approval sub-flow

```
jobs.parts_approval_status: draft → pending → approved | declined
parts_approvals.status:     pending → approved | declined
jobs.parts_requests (jsonb) holds the per-line items
parts_approvals.parts_items (jsonb) snapshots the items at submission
```

The customer-facing public approval lives at `/view-approval` (RW) and is
serviced by the **public, no-JWT** edge functions `get-parts-approval`,
`submit-parts-approval`, `list-customer-parts-approvals`.

### 4.4 Band / polish / precious-metals sub-flow

Driven by `job_components` (one row per physical component being routed to a
non-bench department):

```sql
SELECT department, value, label, is_default, is_active, sort_order
FROM public.process_stages ORDER BY department, sort_order;
```

| Department | Default stage | All stages |
| --- | --- | --- |
| `watchmaker` | `uncased` | uncased → in_progress → in_safe → waiting_components → assigned |
| `band_repair` | `in_progress` | in_progress → in_safe → waiting_components → assigned |
| `precious_metals` | `in_progress` | in_progress → in_safe → waiting_components → assigned |
| `refinishing` | `in_refinishing` | in_refinishing → in_safe → waiting_components → assigned |

`job_sub_components` add a third nesting level (Dial, Hands, Crystal, etc.)
under a `watch_head_component_id`, with `sub_component_types.can_outsource`
gating dispatch to `outsource_dispatches`/`outsource_dispatch_items`.

### 4.5 Component-status snapshot

```sql
SELECT department, status, count(*) FROM public.job_components
GROUP BY department, status ORDER BY department, status;
```

| department | status | count |
| --- | --- | --- |
| band_repair | cancelled | 1 |
| band_repair | in_progress | 9 |
| band_repair | in_queue | 18 |
| band_repair | in_safe | 5 |
| band_repair | waiting_components | 40 |
| refinishing | in_queue | 12 |
| refinishing | waiting_components | 1 |
| watchmaker_bench | in_progress | 154 |
| watchmaker_bench | in_queue | 12 |
| watchmaker_bench | in_safe | 12 |
| watchmaker_bench | waiting_components | 2 |
| *(null dept)* | in_queue | 110 |
| *(null dept)* | waiting_components | 3 |

### 4.6 Service-code → job-type mapping (drives `weeks` → `due_date`)

```sql
SELECT code, job_type, label, weeks, is_movement_service, is_bracelet
FROM public.service_codes WHERE is_active ORDER BY code;
```

| code | job_type | weeks | movement | bracelet |
| --- | --- | ---: | :-: | :-: |
| A   | antique_movement | 27 | ✓ |   |
| A2  | antique_lv2      | 27 | ✓ |   |
| B   | bracelet_repair  |  3 |   | ✓ |
| BR  | bracelet_repair  |  3 |   | ✓ |
| C   | chrono           |  8 | ✓ |   |
| C2  | chrono_lv2       | 12 | ✓ |   |
| CW  | case_work        |  4 |   |   |
| GB  | gold_bracelet    |  6 |   | ✓ |
| M   | modern_movement  |  4 | ✓ |   |
| M2  | modern_lv2       |  4 | ✓ |   |
| P   | case_work        |  4 |   |   |
| SR  | stretch_repair   |  3 |   | ✓ |
| V   | vintage_movement | 12 | ✓ |   |
| V2  | vintage_lv2      | 12 | ✓ |   |
| W   | (routing-only)   |  0 |   |   |
| WR  | warranty         |  4 | ✓ |   |

Department prefixes used in component label IDs (per
`rollisuite-change-order/index.ts`): `W-…` → watchmaker, `BR-…` → band_repair,
`POL-…` → polish, `PM-…` → precious_metals.

---

## 5. Integration back to RolliSuite (RS)

RS project ref `djbjwcoddddywkgljuja`. All RW edge functions live under
`supabase/functions/`. The `rollisuite-*` namespace is RW-side; some are
called *by* RS (inbound webhook), others are called *by RW* into RS (outbound).
All are JWT-disabled in `supabase/config.toml` (verify_jwt=false) and use
`ROLLISUITE_API_KEY` (RS) or `RC_SERVICE_KEY` (customer-portal) for auth.

| Edge function | Direction | What flows | Failure surface |
| --- | --- | --- | --- |
| `rollisuite-webhook` | RS → RW | `WORK_COMPLETED { eventId, eventType, jobId(=estimate_number), salesOrderId, completedAt }`. Marks the matched `jobs` row `finished` | 404 if `jobs.estimate_number` not found (silent: just logs); RS retry behaviour unknown |
| `rollisuite-intake` | RS → RW | New-watch intake payload (estimate, customer, service codes) → creates `customers`/`watches`/`inspections` skeleton in RW. Loads `service_codes` map from DB w/ hardcoded fallback | Optional `fullName` parse; ignores unknown service codes via fallback |
| `rollisuite-change-order` | RS → RW | Estimate edit on RS → reconciles `job_components` (adds/removes by prefix `PM-`/`POL-`/`BR-`/`W-`) | Mismatched prefix → `UNKNOWN` department; logged |
| `rollisuite-so-fulfilled` | RS → RW | Sales-order fulfilled (parts arrived) → updates `parts_approval_status` / job status off `parts_on_order` | Hard-coded RS auth; silent on miss |
| `rollisuite-customer-sync` | RS → RW (or RW pull) | Customer record sync | — |
| `rollisuite-model-sync` | RS → RW | Model/caliber reference updates → `model_references` | — |
| `rollisuite-sync` | RW → RS | Pushes a watch/part-number record on watch creation (via the `sync_watch_to_rollisuite()` Postgres trigger function: posts to `/functions/v1/rollisuite-sync` which forwards to RS) | Fire-and-forget via `net.http_post` |
| `rollisuite-status-push` | RW → RS | `{ estimate_number, status, status_updated_at, finished_date? }`. Status-mapped to `on_hand` for everything except `finished` → `ready_to_ship` | Skipped silently if `ROLLISUITE_API_KEY` missing; otherwise non-blocking |
| `rollisuite-watchmaker-assign` | RW → RS | `{ estimate_number, assigned_watchmaker, assigned_at, status }` whenever `assign_watchmaker()` RPC fires | Fire-and-forget from Postgres `net.http_post` |
| `rollisuite-parts-approved` | RW → RS | Customer-approved parts request → triggers RS sales-order creation | Logs error; does not roll back RW approval state |
| `rollisuite-part-allocation` | RW → RS | Allocation request for a part (non-blocking `x-api-key` POST) | Stock denial does not block the watchmaker |
| `rollisuite-price-lookup` | RW → RS | Catalog price lookup for parts UI | Returns null on RS error |
| `rollisuite-job-finished` | RW → RS | Final "this job is closed in the shop" hand-off | Idempotent on RS side per memory |
| `hit-list-push` | RW → RS | Daily 6 AM EST snapshot of active jobs + jobs finished today, mapped via `STATUS_MAP` to RS-style stages (e.g. `parts_approval`→`parts_pending_approval`, `finished`→`complete`). POSTs to `https://djbjwcoddddywkgljuja.supabase.co/functions/v1/hit-list` | Skipped if no API key; logs but doesn't retry |
| `hit-list` | (RW-internal) | Serves `/daily-hit-list` page; same query shape as the push | — |
| `get-inspection-approval` / `submit-inspection-approval` | Public | T&C-bearing customer page for the inspection sign-off; submit pushes the acceptance to RS as well | Public RLS allows pending-row update only |
| `get-parts-approval` / `submit-parts-approval` | Public | Same shape, for parts | — |
| `list-customer-approvals` / `list-customer-parts-approvals` | RC portal | Read-only structured fetch keyed by customer email, gated by `x-rc-service-key` (`RC_SERVICE_KEY`) | 401 if header missing |
| `ai-assistant` | RW (internal) → RS DB | OpenAI-compatible tool loop. **Photo lookup** crosses projects: builds a `createClient(RS_URL, RS_KEY)` and joins `estimates → client_property → inspections → inspection_photos` on RS, then returns URLs of the form `${RS_BASE}/functions/v1/get-photo?path=…` | RS schema drift will break the chain; logged with `RS inspection_photos lookup error` |

**Dependency direction summary.** RW is downstream of RS for intake and
catalog data, and upstream of RS for status, hit-list, watchmaker
assignment, and customer-approved parts/inspection events. RW's database
does not have RS as a hard dependency at SQL level — every cross-project
read or write goes through HTTP edge functions.

---

## 6. Integration with RolliTime (RT)

**Current state: not wired into RW.**

Evidence:

```bash
rg -i "rollitime|downgrad" supabase/functions src --type ts
# → 0 matches in edge functions; matches in src only refer to internal
#   "downgrade" of UI state, not an external RT seam.
```

There is no edge function named `rollitime-*` and no inbound endpoint that
accepts an RT push. There is also no `process_level` / `downgraded_at`
column on `jobs` or `job_status_changes`. The conceptual flow described —
"RT pushes 'in testing' to RW, then triggers a downgraded process level
(return-to-watchmaker)" — would today have to land on the existing
`set_job_status` RPC with `new_status='in_testing'` and rely on the
watchmaker to manually move the job back to `in_progress`. There is no
audit trail today distinguishing an RT-driven downgrade from a manual one.

**[FILL IN — confirm whether RT exists as a deployed system today or is
planned; if deployed, supply its base URL/project ref and the contract
shape RT will POST to RW]**

**[FILL IN — desired RW endpoint name and payload (e.g. `rollitime-result`
POSTing `{ estimate_number, result: "passed"|"failed", failure_reason?,
tested_at }`) so it can be specced]**

---

## 7. Behavioural data (aggregate)

### 7.1 Live distribution of `jobs.status`

```sql
SELECT status, count(*) FROM public.jobs GROUP BY status ORDER BY count(*) DESC;
```

| status | count |
| --- | ---: |
| finished | 330 |
| in_progress | 75 |
| waiting_approval | 65 |
| in_queue | 31 |
| in_testing | 20 |
| parts_approval | 5 |
| **total** | **526** |

> Note: `intake`, `uncased`, `parts_on_order`, `waiting_components` are
> permitted by `set_job_status` but currently have **zero** live rows. They
> do appear in `job_status_changes`, confirming the pipeline does traverse
> them — they just do not sit there. This matches the architectural claim
> that the lifecycle lives in RW: RS sees the job as "intake" the moment it
> hands it off, and RW moves it forward immediately.

### 7.2 Jobs created per month (last 12)

```sql
SELECT to_char(date_trunc('month', created_at),'YYYY-MM') AS month, count(*)
FROM public.jobs
WHERE created_at > now() - interval '12 months'
GROUP BY 1 ORDER BY 1;
```

| month | count |
| --- | ---: |
| 2026-01 | 158 |
| 2026-02 | 53 |
| 2026-03 | 95 |
| 2026-04 | 73 |
| 2026-05 | 120 |
| 2026-06 | 27 *(partial month)* |

### 7.3 Jobs finished per month (last 12)

```sql
SELECT to_char(date_trunc('month', finished_date),'YYYY-MM') AS month, count(*)
FROM public.jobs
WHERE finished_date > (now() - interval '12 months')::date
GROUP BY 1 ORDER BY 1;
```

| month | count |
| --- | ---: |
| 2026-01 | 19 |
| 2026-02 | 46 |
| 2026-03 | 54 |
| 2026-04 | 68 |
| 2026-05 | 78 |
| 2026-06 | 8 *(partial month)* |

### 7.4 Average time-in-stage (derived from `job_status_changes`)

```sql
WITH seq AS (
  SELECT job_id, new_status, changed_at,
         lead(changed_at) OVER (PARTITION BY job_id ORDER BY changed_at) AS next_at
  FROM public.job_status_changes
)
SELECT new_status,
       count(*) AS transitions,
       round(avg(extract(epoch FROM (next_at - changed_at))/3600)::numeric, 2) AS avg_hours
FROM seq WHERE next_at IS NOT NULL
GROUP BY new_status ORDER BY avg_hours DESC NULLS LAST;
```

| stage entered | transitions out of stage | avg hours in stage |
| --- | ---: | ---: |
| in_progress | 291 | 364.16 |
| finished | 8 | 344.03 *(small sample — re-opens)* |
| in_testing | 184 | 306.94 |
| in_queue | 169 | 296.06 |
| parts_approval | 95 | 169.22 |
| waiting_approval | 297 | 131.97 |
| parts_on_order | 2 | 86.18 |
| uncased | 4 | 5.74 |
| inspection | 1 | 0.01 |

Bottlenecks (≥1 week): **in_progress (~15 days), in_testing (~13 days),
in_queue (~12 days)**.

### 7.5 Watchmaker workload — active jobs (status ≠ finished)

```sql
SELECT coalesce(assigned_watchmaker,'(unassigned)') AS wm, count(*)
FROM public.jobs WHERE status != 'finished' GROUP BY 1 ORDER BY 2 DESC;
```

| watchmaker (initials) | active jobs |
| --- | ---: |
| *(unassigned)* | 102 |
| EG | 43 |
| CS | 36 |
| WG | 11 |
| RC | 3 |
| JV | 1 |

### 7.6 Watchmaker workload — lifetime

```sql
SELECT coalesce(assigned_watchmaker,'(unassigned)') AS wm, count(*)
FROM public.jobs GROUP BY 1 ORDER BY 2 DESC;
```

| watchmaker | lifetime jobs |
| --- | ---: |
| *(unassigned)* | 252 |
| EG | 136 |
| CS | 56 |
| RC | 43 |
| WG | 38 |
| JV | 1 |

### 7.7 Daily hit-list contents and size

The hit-list query is `jobs WHERE status != 'finished' LIMIT 1000` joined
back to inspections/watches/customers, plus today's `finished` jobs (per
`hit-list-push/index.ts`). Aggregate size today:

```sql
SELECT count(*) AS open_jobs FROM public.jobs WHERE status != 'finished';
-- → 196
SELECT count(*) AS overdue FROM public.jobs
WHERE due_date < current_date AND status != 'finished';
-- → 69
SELECT count(*) AS due_dated FROM public.jobs
WHERE due_date IS NOT NULL AND status != 'finished';
-- → 196
```

| metric | value |
| --- | ---: |
| Open jobs (hit-list candidates) | 196 |
| Of which overdue (`due_date < today`) | 69 |
| With due date set | 196 *(100% of open)* |

### 7.8 Other row counts

```sql
SELECT 'jobs', count(*) FROM jobs UNION ALL
SELECT 'inspections', count(*) FROM inspections UNION ALL
SELECT 'inspection_approvals', count(*) FROM inspection_approvals UNION ALL
SELECT 'parts_approvals', count(*) FROM parts_approvals UNION ALL
SELECT 'watches', count(*) FROM watches UNION ALL
SELECT 'customers', count(*) FROM customers UNION ALL
SELECT 'job_components', count(*) FROM job_components UNION ALL
SELECT 'job_status_changes', count(*) FROM job_status_changes UNION ALL
SELECT 'scanner_uploads', count(*) FROM scanner_uploads UNION ALL
SELECT 'liability_waivers', count(*) FROM liability_waivers;
```

| table | rows |
| --- | ---: |
| jobs | 526 |
| inspections | 619 |
| inspection_approvals | 666 |
| parts_approvals | 103 |
| watches | 765 |
| customers | 9458 |
| job_components | 379 |
| job_status_changes | 1547 |
| scanner_uploads | 0 |
| liability_waivers | 29 |

### 7.9 Inspection-approval status mix

```sql
SELECT status, count(*) FROM public.inspection_approvals
GROUP BY status ORDER BY count(*) DESC;
```

| status | count |
| --- | ---: |
| approved | 429 |
| pending | 237 |

```sql
SELECT status, count(*) FROM public.parts_approvals
GROUP BY status ORDER BY count(*) DESC;
```

| status | count |
| --- | ---: |
| approved | 80 |
| pending | 23 |

---

## 8. Photo & inspection handling

### 8.1 Where photos live

| Surface | Storage | RW DB linkage |
| --- | --- | --- |
| **Inspection photos** (dial-removed front/back, movement, etc., captured in the watchmaker room) | **Cloudflare R2 on RolliSuite.** `ai-assistant/index.ts` (lines 207–224, 304–346) confirms: photos are written by RS, indexed in RS's `inspection_photos` table (`id, inspection_id, photo_url, photo_type, photo_order, created_at`), and served through the RS edge function `${RS_BASE}/functions/v1/get-photo?path={storage_path}` | None directly. RW resolves them by joining RS `estimates → client_property → inspections → inspection_photos` keyed by `estimate_number`. |
| **Scantron form scans** (paper inspection sheets) | RW Supabase Storage bucket **`scanner-uploads`** (private). Folder convention: `pending/{timestamp}-{filename}` | `scanner_uploads` table (`id, storage_path, filename, status, processed_at, created_at`) — currently empty (0 rows). The `scan-scantron` edge function consumes them and OCRs via Gemini. |
| **Scantron blank templates** | RW Supabase Storage bucket **`scantron-templates`** (public) | `scantron_templates` (`id, version, label, storage_path, is_active, uploaded_by, created_at`) |

### 8.2 Inspection capture flow inside RW

1. Manager/owner opens `/inspections/new` for a `watches` row.
2. `InspectionForm.tsx` collects per-component condition (each rendered
   as a jsonb column on `inspections`: `dial_condition`, `hands_condition`,
   `bezel_condition`, `crown_condition`, `case_condition`, `crystal_condition`,
   `bracelet_condition`), plus `notes`, `job_types`, `total_estimate`,
   `pricing_details`.
3. Optional Scantron path: physical form is scanned at the bench →
   uploaded to `scanner-uploads` via `scanner-upload` edge function → OCR'd
   by `scan-scantron` (Gemini Vision) → returns inspection items prefilled
   for review.
4. Customer review: `send-inspection-email` posts a public URL to the
   customer, who interacts with `inspection_approvals` (pending → approved/
   declined) via `get-inspection-approval` / `submit-inspection-approval`.
5. The photo lookup chain (§8.1, RS side) is what feeds the future
   "dial-photo library / authenticator" the AI assistant exposes via the
   `get_inspection_photos` tool.

### 8.3 What's missing for a dial-photo library inside RW

- No RW table currently stores photo metadata (filename, taken_at, watch_id,
  photo_type). Every read goes back across the seam to RS.
- No RW-side R2 client or credentials.
- **[FILL IN — whether the desired dial-photo authenticator should live in
  RW (mirror the metadata locally) or remain a thin proxy over RS]**

---

## 9. Open questions / `[FILL IN]` index

| # | Section | Placeholder |
| --- | --- | --- |
| 1 | §1 System overview | Legal/business identity behind RolliWorks (entity, location, ownership relationship to the RS business) |
| 2 | §1 System overview | Headcount of bench watchmakers vs. band/polish techs vs. front office expected to use RW day-to-day |
| 3 | §6 RolliTime | Confirm whether RT exists today or is planned; supply base URL / project ref if deployed |
| 4 | §6 RolliTime | Desired RW endpoint name + payload shape for RT push (e.g. `rollitime-result`) |
| 5 | §8.3 Photos | Whether the dial-photo authenticator should mirror photo metadata into RW or remain a thin proxy over RS |

---

*End of dossier. All metrics above were produced by SELECT-only queries
against project `pkgnrcfqrldwjibghefm` on 2026-06-07. No DML, DDL, or other
file edits were performed during generation.*
