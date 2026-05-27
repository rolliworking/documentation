---
silo: 7
date: 2026-05-14
status: partial — multiple fixes shipped to test branches, validation pending
last_validated: 2026-05-14 evening session
---

# Silo 7

## 1. Identification

**Main topic:** Catching and fixing real bugs while exploring system architecture. Specifically:
- Performance Verification Report: add `client_first_name` capture and display
- SO duplicate-line bug root cause investigation (two independent defects identified)
- Shop floor QC inspect drag crash (DB CHECK constraint violation)
- Manager `/shop-floor` access — turned out to be a non-bug, but produced full permissions architecture documentation
- Job History Lookup: design investigation for shop floor data visibility (not built)
- Handoff document rewritten end-of-session to capture silo 6 + 7 gains

**Date range:** May 14, 2026 (evening into night, single session).

**Status at chat close:** Multiple fixes applied to test branches, partial verification, several items deferred for next session. Handoff doc rewritten to capture session's gains. Silo packaging in progress.

---

## 2. What shipped

### Verified live (RW test branch — user confirmed working in-session)

- **B1+B2: `station_id` column + drag routing fix.**
  - Migration: added `station_id text` nullable column to both `job_components` and `job_component_logs` (no defaults, no backfill).
  - `STATIONS[]` entry for qc_inspect changed: `department: "qc"` → `department: "band_repair"` (constraint-legal).
  - Patched BOTH commit handlers in `ShopFloor.tsx`:
    - `handleCommit` (lines 995-1060) — scan-to-station queue path
    - `commitDrop` (lines 645-689) — drag-drop + TechPickerDialog path
  - Both now write `station_id = station.id` to the component update AND to the log insert.
  - `dotStationFor()` refactored (lines 155-203): checks station_id first, returns it if valid (`STATION_BY_ID.has(stationId)`), falls through to existing logic for NULL or unknown values.
  - Dead branch removed: `if (department === "qc") return "qc_inspect"` (unreachable after constraint-legal rewrite).
  - LookupDot select expanded to include `station_id`; passed to `dotStationFor()`.
  - **Verified:** migration ran (confirmed via `information_schema.columns` query returning two rows). User confirmed "its working" — QC inspect drag succeeds, dot persists at QC across page refreshes.

### Applied to RS test branch (Lovable confirmed Apply, partial verification)

- **Performance Report `client_first_name` capture and display.**
  - Migration: `ALTER TABLE performance_verification_reports ADD COLUMN client_first_name text` (nullable).
  - `useSavePerformanceReport` payload interface extended to include `client_first_name`.
  - `PerformanceReportEntryPage.tsx`: `IdentityState` extended, `emptyIdentity()` updated, SO prefill query extended to pull `customers(first_name, last_name)`, hydration paths updated (both SO and existing-report branches), save payload updated, new "Client First Name" form field added immediately before "Client Last Name".
  - `PerformanceReportPreviewPage.tsx`: IdentityTable Client row renders `[r.client_first_name, r.client_last_name].filter(Boolean).join(' ') || '—'` — graceful NULL fallback for historical rows.
  - **Apply state:** User stated "approved it" (typed as "aiiproved it"). Migration existence NOT directly verified by SQL in-session. Smoke test on real SO with populated customer first_name NOT yet performed.

- **Fix A: persistOrder UUID patch-back.**
  - `persistOrder` INSERT branch modified to use `.insert(...).select()` and patch local `lineItems` state by position so subsequent saves match `existingLineMap` (preventing the non-UUID-id re-INSERT pattern).
  - **Apply state:** User stated "Applied" and ran Test 1 (save → save → refresh on test branch): passed. Test 2 (with navigation: save → navigate to perf report → return → save → refresh) NOT yet performed. Test branch only — not merged to main.

### Not shipped (designed/scoped but not applied)

- **Fix B: `handleQuickBarcodeSubmit` re-entry guard.**
  - One-line fix: `if (isQuickSearching) return;` at top of handler.
  - Prompt drafted at end of session but user pivoted to silo packaging before pasting to Lovable.

---

## 3. What was learned (the texture)

### SO duplicate-line root cause — TWO independent bugs

Forensic evidence: SO 501694 has 28 lines, 16 distinct, 12 duplicate rows. Two write batches at 22:23:33 (10 rows, includes one in-batch dup of 2130-540) and 22:39:25 (~16 min later, the same 10 rows re-inserted with new UUIDs). Part 2130-540 appears 4 times total.

**Bug 1 (in-batch dup, Write 2 pattern):** `handleQuickBarcodeSubmit` in `SalesOrderDetailPage.tsx` (lines 525-704) is async, performs ~600ms of Supabase round-trips, and sets `isQuickSearching=true` for UI button disable. But the `<form onSubmit>` still fires on Enter regardless of button state, and the handler never reads `isQuickSearching` as a re-entry guard. Second Enter during the async window starts a parallel handler, both find the same part, both append `[..., newItem, blankLine]`. Same part lands at two sort_orders in one save. Session replay confirmed pattern (Part 206C, Recut Fluted bezel, Bezel Laser Welding all appear as 2 copies in the same batch — consistent with rapid Enter-presses).

**Bug 2 (post-navigation dup, Write 3 pattern):** `addLineItem` (line ~431) and quick-add path use `String(Date.now())` as `id`. After `persistOrder` INSERTs those rows to DB, the assigned DB UUID is never written back into local state. On any subsequent save, `existingLineMap.get("1715712345")` returns undefined. The asymmetric guard at lines 784-796 only catches UUID-shaped local ids without DB match — non-UUID ids fall through to `linesToInsert` and re-INSERT unconditionally. Combined with `so-draft-${id}` localStorage restore (`draft.savedAt > order.updated_at` overlays draft state on freshly-hydrated DB lineItems), every navigation that doesn't bump `sales_orders.updated_at` triggers re-insert on the next save. Performance Report saves don't bump `sales_orders.updated_at` (no trigger from `performance_verification_reports` to `sales_orders`), making perf-report navigation a perfect trigger.

User's prior framing ("the Performance Report adds line items to the SO") was wrong — perf report doesn't touch `so_lines` at all. It's just the navigation trigger that exposes Bug 2.

### RW permissions architecture (previously undocumented)

Two parallel access-control systems:

1. **Permission-key system** (route/nav gating): `RequirePermission` component reads `useUserPermissions()` and calls `hasPermission(userPermissions, key)`. Routes in `routes.tsx` declare `permission: "<key>"`. `NavItems.tsx` uses the same key to hide nav items. Source of truth: `public.role_permissions(role, permission_key)`.

2. **Role-string system** (in-page feature gating): `useUserRole()` hook reads `public.user_roles.role`. Used directly in component code, e.g. `const isAdmin = role === "owner" || role === "manager";`.

**Roles vocabulary: `owner`, `manager`, `staff`. There is no `admin` role.** Mike's verbal usage of "admin" maps to `owner`.

Manager has 25 of 28 owner permissions. The 4 owner-only keys: `data.export`, `users.manage`, `users.permissions`, `users.view`. Manager has `jobs.view`, which is what gates `/shop-floor`. **Managers can access `/shop-floor` today.**

Edge case: a user with no `user_roles` row defaults to `staff` in `useUserRole()` (per `use-user-role.ts:21`), but `useUserPermissions()` returns `[]` (empty array), which blocks them entirely via `RequirePermission`. Users created via auth-only flows must have a `user_roles` row inserted or they cannot access anything.

### Shop floor commit asymmetry

`ShopFloor.tsx` has TWO commit handlers for the same kinds of writes:

1. **`commitDrop`** (lines 645-689) — fires from drag-drop + TechPickerDialog (the drag flow)
2. **`handleCommit`** (lines 995-1060) — fires from scan-to-station queue (barcode scanner panel)

These handlers have similar but not identical code (different update object shape, different log insert fields). Changes that affect "what gets written when a dot moves" need BOTH paths updated. The first silo 7 attempt at B1 only patched `handleCommit`; the drag flow goes through `commitDrop`, which is why station_id wasn't being written from drag. Required B2 follow-up.

Tech debt: should be unified into a single helper.

**Adjacent finding:** `commitDrop` hardcodes `assigned_to: null` in its log insert (line 679). `handleCommit` correctly writes `tech?.id`. This means timelines reconstructed from drag-flow log rows cannot identify the tech who was assigned — only that someone was. Worth fixing if Job History Lookup ever surfaces shop floor events.

### `job_components.department` CHECK constraint

Allowed values: `watchmaker_bench`, `refinishing`, `band_repair`, `precious_metals`, `safe_storage`.

The qc_inspect station previously wrote `department: "qc"`, which is NOT in the whitelist. Postgres rejected every drag-to-QC attempt with `new row for relation "job_components" violates check constraint "job_components_department_check"`. Fix: write `band_repair` instead, and route via `station_id` (silo 7).

### Performance Report architecture

Identity fields on `performance_verification_reports` (brand, model, ref, serial, client names, estimate_number) are **text snapshots, not joins**. Preview never re-fetches. Snapshot drift hazard if source data is corrected post-report.

- No unique constraint on `sales_order_id` — multiple reports per SO physically allowed.
- Lookup returns most recent via `order created_at desc limit 1`.
- No list page to find orphan reports.
- No email/send action despite branded customer-facing styling. Staff downloads JPG/PDF and attaches manually.
- Single hard-coded variant. All copy lives inline in `PerformanceReportPreviewPage.tsx`. No `report_type` column.
- Ref-serial rendered as `"<ref> / <serial>"` (space-slash-space) — different from RW canonical `<ref>-<serial-last-3>` format. Different conventions are intentional/acceptable per Mike.

### Job History Lookup current state

- File: `src/components/jobs/JobHistoryLookup.tsx` (706 lines, default export).
- Single entry point: sidebar "Job History" button in `AppShell.tsx:73`.
- Dialog component, fetched once per selection (not live).
- `buildTimeline()` (line 95) merges 4+ data sources into `TimelineEvent[]`, sorts descending.
- Currently surfaces: job created/intake/work_started/finished, inspection completion, client approvals from `inspection_approvals`, parts requests, sent emails. **Does NOT surface any shop floor activity.**
- Adjacent unused data sources: `job_status_changes` (parent job transitions), `job_sub_component_logs` (Dial/Hands/Crystal status).
- 92 of 135 historical log rows have NULL `station_id` (pre-B1) — any UI must handle gracefully.

---

## 4. What was deferred

- **Fix B (handleQuickBarcodeSubmit re-entry guard).** One-line fix. Scoped, prompt drafted, not yet applied. Independent of Fix A. Should ship next session.
- **Fix A test 2.** Save → navigate → return → save → refresh, verify no dups. Pending real order on test branch.
- **Performance Report smoke test.** Open SO with customer first_name, verify auto-populate. Open historical report, verify graceful NULL fallback in preview.
- **Migration verification SQL** for Performance Report `client_first_name`. Not run by user in-session.
- **Job History Lookup shop floor visibility.** Investigation complete, design recommended (per-component current-state block at dialog top). Not built. Recommended for fresh chat next session — design work benefits from full context budget.
- **`commitDrop` `assigned_to` fix.** Hardcoded null limits timeline fidelity. Prerequisite for any future event-timeline view in Job History Lookup.
- **`commitDrop` / `handleCommit` unification.** Tech debt.
- **Auto-assign single-tech stations.** Mike asked about skipping picker for QC inspect (JV is primary QC tech). Real product call. Open questions: only QC tech ever, or just primary? Apply pattern to other single-tech stations? Handle inactive tech (`is_active = false`)?
- **Plan A validation.** Tomorrow's first intakes verify silo 6 work (band-only payload field 11 = 1, field 12 = 0; etc.).
- **Type cast cleanup.** `as { station_id: string | null }` cast added "temporarily" in SalesOrderDetailPage pending types regen; should be removed when noticed.

---

## 5. What was wrong

- **Mike's initial framing "managers can't access /shop-floor"** turned out to be a non-bug. Investigation showed manager role already has `jobs.view` permission and the route only checks that key. Mike later confirmed: "it was a browser issue. manager level can see dots." Real outcome of pursuing the false symptom: documented the entire permissions architecture, which had been undocumented before. Net positive.

- **Sent RS performance-report investigation prompt to RW Lovable** — Mike pasted into the wrong window ("oops toot many windows open"). RW Lovable correctly refused rather than hallucinating ("the previous investigation referenced in your prompt is not part of this project"). Good outcome (no hallucination) but reinforces the "label prompts clearly, check window focus" failure mode.

- **B1 first attempt missed `commitDrop`.** Investigation found `handleCommit` and stopped. B1 shipped with only one of two drag handlers patched. User testing revealed drag-to-QC still routed back to band_tech on next render. Diagnostic SQL query showed `station_id IS NULL` on the dragged row — write side never fired. B2 (two-line patch to `commitDrop`) followed in the same session. Lesson: shared-code investigations need explicit "find ALL handlers that touch this surface, including async dialog flows" instruction. Documented in handoff Section 8 ("Lovable searched but missed code paths" pattern).

- **Initial assumption that Performance Report flow wrote to `so_lines`.** Mike's framing: "extras appear after the performance report is done." Investigation proved the report flow only writes to `performance_verification_reports`, never touches `so_lines`. The report is the navigation trigger that exposes the bug, not the source of the writes. Real root cause was the persistOrder + draft-restore interaction (Bug 2 above).

- **Scope creep nearly added ref-serial format change.** Mike asked for "customer name and ref-serial#" on Performance Report. Initial scoping included a "fix" to the ref-serial format (`/` → `-`, etc.). Mike pushed back hard: "why are we changing or asking about this. leave the report as-is." Correctly scoped down to first-name addition only. Lesson: when the user says "needs X and Y," verify whether X and Y are missing or present-but-suspect before designing fixes.

- **Multiple "Applied" claims that needed verification.** Lovable's response language doesn't reliably distinguish plan-mode from apply-mode. At least three times in silo 7, "Applied" or "edits applied" claims required follow-up verification (git status, direct DB query, or running the actual code). One Lovable plan ended mid-sentence at "Out of sco" — clipboard truncation suspected, follow-up needed.

- **Time-pressure miscommunication.** Recommended stopping/parking work at multiple points based on assumption Mike was deep into a long session. Mike eventually clarified "it's 11pm" — fewer hours of fatigue accumulated than recommendation assumed. Adjustment: factor stated session length into time-pressure reasoning, not just turn count.

---

## 6. Key decisions

### Fix A approach: UUID patch-back over guard widening

Two viable approaches existed:
- **Approach 1 (chosen):** Patch local `lineItems` state with assigned UUIDs after INSERT. Invisible to user.
- **Approach 2:** Widen the asymmetric guard to throw on ANY non-matching local id (not just UUID-shaped). Forces user to refresh before saving.

Picked Approach 1 because it addresses the root cause (local state never gets the DB UUID) rather than surfacing the failure mode as a user-visible error. Conditions to reconsider: if patch-back has unexpected interactions with autosave (writing patched UUIDs to draft, draft restored on remount with correct ids matching DB) or with concurrent edits (user modifying line between INSERT firing and response arriving).

### Performance Report: Option A (first-name only) over Option B (full identity overhaul)

Mike confirmed: "we still use client name for business clients as we need a contact person. options A but last is only stays in place." So:
- `client_last_name` column and form field stay exactly as-is.
- Add `client_first_name` as additive nullable column + form field + preview render.
- No `display_name` or `company_name` field.
- No change to existing customer last-name plumbing.
- Graceful NULL fallback for historical rows (filter(Boolean).join handles empty/null).

### QC inspect: B1 (schema column) over A (crash-only) or B2 (composite signal)

Three options for fixing the QC inspect crash + routing:
- **A:** Change `department: "qc"` → `band_repair`, accept that dot will route to `band_tech` on next render (cosmetic dot-escape bug).
- **B1 (chosen):** Add `station_id text` column to `job_components`, write it on every drag, read it first in `dotStationFor()`.
- **B2:** Composite signal — match on `department = 'band_repair' AND assignee in QC tech list`. Requires identifying QC techs.

Picked B1 because:
1. Dot position is real persisted state — survives sessions, refreshes, devices. Should be in DB.
2. Future stations (Chunks 4/5) benefit from same column. Adding it now cheaper than later.
3. `dotStationFor()` already has complex fallback chain. Reading `station_id` first short-circuits cleanly.
4. Investigation confirmed ZERO ripple to other writers — `station_id` stays localized to `ShopFloor.tsx`. No other code path can produce QC routing intent.

Conditions to reconsider: if multiple writers in the future need to set `station_id` (e.g. Bulk Assign or scanner adds a QC routing path), the model still works — just needs more places to write it.

### Park new work to write handoff late in session

User had ~2.4 hours of clock remaining but session had produced significant work product (multiple fixes shipped to test, multiple architectural findings) that wasn't yet in the durable handoff doc. Recommended pausing new feature work, writing handoff update first.

Reasoning: context budget was approaching limits, and the cost of "we shipped six items but lost three of them when context died" exceeds the value of one more ship. User accepted; handoff written; Fix B prompt drafted; Job History Lookup deferred to fresh chat.

---

## 7. Artifacts worth preserving

See `fixtures/` directory for detailed records of:

- **SO 501694** (RS) — Canonical Fix A / Fix B test target. 28 lines, 16 distinct, 12 duplicates. Two write batches.
- **EST-24392 / Larissa Oprysk** (RW) — QC inspect crash test bed. Used as fixture for Job History Lookup investigation.
- **EST-12054 / Lee Mitchell, SO 501690** (RS) — Original "different versions of same SO" complaint that led to draft-restore architecture investigation.
- **EST-24027 / Jenny Davis** (RW) — "No components on this job" + backfill screenshot. Used to verify backfill "+" still works under manager role and shop floor renders for non-owner accounts.

See `fixtures/diagnostic-sql-queries.md` for SQL queries that produced useful answers during this silo.
