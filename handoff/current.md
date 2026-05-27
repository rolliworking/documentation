## Silo 6 contributions (2026-05-12 to 2026-05-14)

### Verified shipped

- set_job_status SQL fix (qualified job_id)
- job_component_notes table (RW database, RLS gated on jobs.view)
- See `silos/silo-6.md` "Verified live" section

### Apply unverified (Lovable confirmed but plan-mode confusion surfaced late session)

- Shop floor geometry fix (VB_W 1240→1260)
- Bulk Assign watch-label scan fix (uses resolveJobFromScan)
- Backfill components "+" feature
- Job lookup popup scroll fix
- Plan A intake payload fixes
- Shop floor drag v1 (7 lock stations)
- Tech picker v2 (5 work-node stations + TechPickerDialog)
- dotStationFor override fix

**ACTION ITEM:** Verify each "apply unverified" item against production code in git. Either confirm shipped or move to backlog as pending-apply.

### Architectural findings (load-bearing)

- QBO push intentionally bypasses set_job_status (raw UPDATE via service role)
- Two parallel state machines (jobs.status, job_components.status)
- "Documented but unused" pattern recurring in codebase
- RS doesn't persist outbound RW intake payloads
- received_case was structurally hard-coded (fixed in Plan A)
- jobs.serial_number actually stores reference numbers (misnamed column)
- job_component_logs.assigned_to is uuid-typed but tech IDs aren't auth UUIDs

### New gaps documented

- `known-gaps/plan-vs-apply-confusion.md`
- `known-gaps/intake-payload-contract-drift.md`
- `known-gaps/rs-no-outbound-audit-log.md`
- `known-gaps/assigned-to-uuid-mismatch.md`

### Plans designed but not started

- Plan B: rw_intake_log table in RS (audit log for outbound payloads)
- Plan C: Structural intake changes (Stage-2 booleans, Confirm & Send UI, payload v2)

### Workflow conventions established

- "Surgical fix" framing = Mike's Approve trigger
- "Mode: Surgical investigation, NO code changes" for read-only investigation
- Plan-vs-apply distinction must be verified, not inferred
- Three-AI cross-check pattern (Claude + RW Lovable + RS Lovable + Mike as bus)

### Open questions

- Apply state of Plan A and shop floor drag/picker items
- EST-24187 approval mismatch (Mike says approved, DB says waiting_approval)
- Has band-only intake catch-up batch happened?
- Does RW caret-delimited parser tolerate unknown trailing fields? (Plan C blocker)

---
silo: 7
date: 2026-05-14
status: append-only contribution to handoff/current.md
last_validated: 2026-05-14
---

# Silo 7 contribution to handoff/current.md

(This section appends to whatever already exists in `handoff/current.md`. If the file is new, this is the seed.)

## From silo 7 (2026-05-14)

### Shipped to RW test branch (verified working)

- **B1+B2: shop floor `station_id` column + drag routing fix.** Added `station_id text` nullable column to `job_components` and `job_component_logs`. Changed qc_inspect station from `department: "qc"` (CHECK constraint violation) to `department: "band_repair"`. Patched BOTH `commitDrop` (drag handler) and `handleCommit` (scan handler) in `src/pages/ShopFloor.tsx` to write `station_id`. Refactored `dotStationFor()` to check station_id first, fall back to existing logic. User-verified working â€” drag-to-QC-inspect now succeeds and persists across refresh.

### Applied to RS test branch (smoke test pending)

- **Performance Report `client_first_name`.** Added nullable column to `performance_verification_reports`. Form field, prefill from `customers.first_name`, preview render with graceful NULL fallback. Migration existence not directly verified by SQL â€” recommended check before merge: `SELECT column_name FROM information_schema.columns WHERE table_name = 'performance_verification_reports' AND column_name = 'client_first_name';`. Smoke test on real SO pending.

- **Fix A: persistOrder UUID patch-back.** `persistOrder` INSERT branch now captures returned UUIDs via `.select()` and patches local `lineItems` so subsequent saves match `existingLineMap`. Test 1 (saveâ†’saveâ†’refresh) passed on test branch. Test 2 (with navigation through Performance Report or similar â€” the SO 501694 production failure mode) pending real order.

### Scoped but not applied

- **Fix B: `handleQuickBarcodeSubmit` re-entry guard.** One-line fix: `if (isQuickSearching) return;` at top of handler in `SalesOrderDetailPage.tsx` lines 525-704. Stops in-batch dups (Write 2 pattern on SO 501694). Prompt drafted, not yet applied.

### Architecture findings now documented (previously undocumented)

- **RW has two parallel access-control systems:** permission-key (`RequirePermission` + `role_permissions` table) for routes/nav, role-string (`useUserRole()`) for in-page features. Roles are `owner`/`manager`/`staff` â€” **no `admin` role exists**. Manager has 25 of 28 owner permissions (missing only `data.export`, `users.manage`, `users.permissions`, `users.view`). Managers CAN access `/shop-floor` today.

- **Shop floor commit asymmetry:** `ShopFloor.tsx` has TWO commit handlers (`commitDrop` for drag, `handleCommit` for scan). Similar but not identical code. Changes touching "what gets written on dot move" need BOTH patched. Tech debt: unify into shared helper.

- **Performance Report identity fields are text snapshots, not joins.** Snapshot drift hazard. Documented in known-gaps.

- **SO duplicate-line bug has TWO independent causes:**
  1. `handleQuickBarcodeSubmit` async re-entry (Fix B target)
  2. `persistOrder` non-UUID local ids never patched back to UUIDs after INSERT (Fix A target, applied)

### Validation pending next session

- Plan A (silo 6) tomorrow morning's intakes
- Fix A test 2 on next real order with navigation
- Performance Report client_first_name smoke test
- B1+B2 verify all station types (band_tech, movement_service, refinish_case, lock stations) on test branch
- Migration existence check for Performance Report client_first_name

### Deferred to next session

- Fix B (5-min one-line fix, prompt drafted)
- Job History Lookup shop floor visibility (designed: per-component current-state block at dialog top; recommend fresh chat for design conversation budget)
- Auto-assign single-tech stations (open product questions about QC tech identification)
- `commitDrop` `assigned_to` fix (prerequisite for event-timeline view in Job History Lookup)

