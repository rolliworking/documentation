---
silo: 6
date_range: 2026-05-12 to 2026-05-14
status: mixed (some shipped verified, some plan-mode unverified)
last_validated: 2026-05-14
---

# Silo 6 — Shop Floor v1/v2 + Plan A Intake Payload Fixes

## Identification

- **Silo:** 6
- **Main topic:** Shop floor drag build (v1 safes, v2 work-nodes with tech picker, dotStationFor render fix) + Plan A intake payload corrections (received_case hardcode, received_bracelet derivation, 24185-class bracelet swap)
- **Date range:** May 12, 2026 evening → May 14, 2026 morning (with 8hr sleep break)
- **Status at close:** Mixed. Some verified live, some plan-mode pending. See "What shipped" below.

## What shipped

### Verified live (high confidence)

- **set_job_status ambiguous job_id fix** — SQL function corrected to qualify `WHERE jc.job_id = set_job_status.job_id`. Verified via `pg_get_functiondef` showing qualified clause live in database.
- **job_component_notes table created** — Migration applied via Supabase SQL editor. Verified via `pg_tables` + `pg_policies` queries showing table exists with RLS + 2 policies (SELECT, INSERT).
  - Migration file: `supabase/migrations/20260513050118_add_job_component_notes.sql`

### Likely shipped (Lovable confirmed, not independently verified)

- Shop floor geometry fix: `VB_W` 1240 → 1260 in ShopFloor.tsx line 41
- Bulk Assign watch-label scan fix: refactored to use shared `resolveJobFromScan`, interface gained `serviceType`/`status`/`inspectionId`
- Backfill components "+" feature: Component Lookup gains "+" button for owner/manager when components.length === 0. Lovable claimed end-to-end verification on est 24800 preview.
- Job lookup popup scroll fix: DialogContent restructured with flex/overflow

### Unknown apply state (plan-mode confusion surfaced late session)

- Plan A intake payload fixes (received_case, received_bracelet, 24185-class bracelet swap)
- Shop floor drag v1 (pointer events, 7 lock stations as drop targets)
- Tech picker v2 (TechPickerDialog + 5 work-node stations: movement_service, refinish_case, refinish_band, band_tech, qc_inspect)
- dotStationFor override fix (wrapping override in `if (status === "in_queue")`)

**Note on apply confusion:** Mike clarified late in session that he only clicks Lovable's Approve button when prompts are framed as "Surgical fix." Real-world test feedback (24187 screenshot showing in-progress bracelet pinned to lock_pre_approval, then unsticking after fix) suggests drag/picker/dotStationFor WERE applied. Plan A apply status unverified.

**Branch:** `test` (Lovable internal edit branches → GitHub sync → test branch)

**Commits/hashes:** Not visible in chat.

## What was learned (texture)

### Architectural findings (load-bearing)

- **QBO push intentionally bypasses set_job_status.** `rollisuite-job-finished` edge function does raw `UPDATE jobs SET status='finished'` via service role key. Bypasses component-readiness check, job_status_changes audit, RS webhook, timestamp stamping. Confirmed correct by Mike. Implication: set_job_status safety check only protects MANUAL finishing.

- **Two parallel state machines.** `jobs.status` (owned by WorkQueue, WmRoom, QBO push) and `job_components.status` (owned by shop floor, BandKanban, ComponentStatus) are independent. Sync only at set_job_status finished-transition check and at StationScanner (unused).

- **"Documented but unused" code pattern.** Recurring throughout RW. StationScanner, BandKanban (zero drag code despite name), componentStatusMap, `final_assembly` station, several work-node stations missing componentStatus/department. Treat code as proposal, verify with operator.

- **RS does not persist outbound RW intake payloads.** No rw_sync_log table. Fire-and-forget from browser. Cannot reconstruct historical payloads. Plan B addresses.

- **Documented payload contract drifted from actual code.** Data Trickle infographic (May 10) listed serial_number, flow, orphan_parked, label_ids as required. Actual code omits these. Plus actual code sends extra fields (email, phone, part_number, date, bracelet_model) not in contract.

- **received_case was hard-coded `true`** in IntakeLabelWizard.tsx:523. Every band-only intake lied about case receipt for entire app history. Fixed in Plan A.

- **received_bracelet was wrong for band-only flows.** ClientWatchEntryPage set it as `itemType === 'watch'` (false for band_only). ReceivePackagesPage didn't pass it at all. Both files needed Plan A fixes.

- **Multi-item bracelet swap (24185-class).** Per-item effect in ClientWatchEntryPage didn't pre-fill formData.bracelet_description in multi-item mode. Operators typed each item's description manually in arbitrary order, causing client_property rows to carry data belonging to wrong estimate lines. Plan A pre-fills from estimate sort_order.

- **jobs.serial_number actually stores reference numbers.** Column misnamed. Don't add new serial_number fallback logic — it's the same as reference_number lookup.

- **job_component_logs.assigned_to is uuid-typed but watchmaker/dept_tech IDs aren't UUIDs.** Existing pattern writes null; relies on assignee_name/initials + notes for audit.

- **dotStationFor() job-status override was too aggressive.** lock_pre_approval/lock_pre_queue fired for ANY component on a job in those statuses. After drag set component to in_progress, override kept dot pinned at lock. Fix gates override on `status === "in_queue"`.

### Operational vs code reality

- "Watch head" = head + case, no bracelet. NOT just movement.
- Mating components is implicit via QBO push, not explicit workflow step. `final_assembly` is vestigial.
- Workers DO start work before formal approval. Old override prevented showing this.
- Shop floor drag is mostly a CORRECTION TOOL for bulk-assign bugs, not primary routing UI.
- Band techs do QC (no separate QC role in department_technicians).
- Tech assignments are sticky once made; reassignment via WorkQueue only.
- Daily Hit List handles stale jobs — shop floor doesn't need aging-color overlays.

### Methodology that worked

- **Investigation-first.** Before designing defensive scenarios, run read-only count queries. All four Chunk-2 scenarios returned 0 → saved the chunk.
- **Three-AI convergence.** Claude (zip) + RW Lovable (codebase) + RS Lovable (codebase) independently identifying same bug = very high confidence. Worked for QBO bypass.
- **Plan-first pattern.** Every Lovable build asked to show plan before applying. Caught hallucinated imports, abbreviated diffs with `...`, invented service markers.
- **"Surgical fix" framing.** Reduces Lovable scope creep + signals Mike's approval intent.
- **Push back on confident-but-wrong assertions.** Lovable hallucinated import, fabricated "porting" of nonexistent fallback, abbreviated diffs hiding real code. Each caught by pushback.

## What was deferred

- StationScanner deletion (after shop floor v1 stabilizes)
- useUpdateJob bypass hardening (permissions + column-level trigger)
- Idempotency keys on RS job_activity_log
- Timestamp clamping on RS inbound webhooks
- RW SQL function param naming convention (`p_` or `_` prefix)
- BandKanban deprecation decision
- ComponentStatus.tsx direct UPDATE writes audit (lines 675, 1013)
- EST-11982 3-week-old draft SO anomaly (separate Cowan issue)
- final_assembly station removal (future cleanup)
- BUG-BACKFILL-FIRST-CLICK (first click sometimes misses after panel viewport shift)
- Aging-color overlay on dots (rejected — Daily Hit List handles)
- Shop Floor Chunk 4: right-click notes UI (consumes job_component_notes table)
- Shop Floor Chunk 5: detail panel on dot click
- Touch support for shop floor drag
- Validate watch-label scan in production (16800-R838612 never tested live)
- Plan B: rw_intake_log table in RS
- Plan C: Structural intake changes (Stage-2 booleans, Confirm & Send UI, payload v2)

**Open questions:**

- Which "applied" Lovable responses resulted in actual Approve clicks? Need per-item verification against production code.
- EST-24187: Mike said "approved" but jobs.status = 'waiting_approval'. Why didn't approval transition the job?
- Has band-only intake catch-up batch happened? Plan A is forward-only; historical client_property rows may carry swapped descriptions.
- Does RW's caret-delimited parser tolerate unknown trailing fields? Plan C v2 payload depends on this.

## What was wrong

### Theories that turned out wrong

- **"Band-only jobs are silently stuck."** Worry that BAND_ONLY_FLOW couldn't finish due to safety check. QBO push bypass meant they were fine.
- **"Orphan job_components rows accumulate."** Chunk 2 scoped for defensive handling. Investigation: zero orphans, zero null stations, zero cancelled components. Chunk 2 was theoretical.
- **"BandKanban uses @dnd-kit."** Zero drag code in BandKanban. Native HTML5 drag only in old ComponentStatus.tsx (broken on SVG/touch).
- **"resolveJobFromScan is already imported in BulkAssign.tsx line 13."** Lovable hallucinated. Actual line 13 imports Input.
- **"watch_head_only intake should produce received_case = 0."** Verification spec wrong. "Watch head" = head + case in this shop. Code was correct.
- **"final_assembly is a real workflow step."** Mike confirmed: vestigial.
- **"Plan A only needs received_case fix."** Lovable's static trace caught received_bracelet had same bug in two files. Plan expanded mid-flight.
- **"Lovable's 'Files saved: yes' means files were saved."** Plan mode produces identical confirmation. Only green Approve button click applies. Discovered late.

### False summits

- "Shipped today: 8" celebrations when items were plan-mode pending
- "Chunk 1 piece 1 done" after Lovable said "Migration written" — table didn't exist until SQL was manually run
- "Plan A approved" after receivedCase clarification — turned out receivedBracelet bug existed too

### Hours wasted

- ~30-40 minutes verifying set_job_status fix when it was already applied earlier in day (could have just asked)
- Multiple cycles of confused Supabase SQL editor vs Lovable due to clipboard truncation
- Initial Chunk 3 design assumed @dnd-kit — restart from scratch after investigation

## Key decisions

See `decisions/*.md` for full reasoning. Brief list:

- **Skip Chunk 2 entirely** — zero defensive cases exist in production
- **Pointer events for drag** — not @dnd-kit (not installed) or HTML5 (broken on SVG)
- **dotStationFor override gated on `status === "in_queue"`** — preserves "not started" signal, lets work-in-progress dots move
- **Defer tech picker from drag v1** — smaller ship first, v2 for picker
- **qc_inspect picker uses band_repair pool** — band techs do QC operationally

## Artifacts worth preserving

See `fixtures/*.md`. Brief list:

- **est-24800** — Component-less job (Rolex Two Tone D Link Jubilee, Michael Test). Used to validate backfill "+" feature.
- **est-24185** — Multi-item band-only with two distinct bracelets. Exposed the bracelet swap bug.
- **est-24187** — In-progress bracelet (assigned to Joseph) on a waiting_approval job. Exposed the dotStationFor override bug.

### Useful diagnostic queries

```sql
-- Verify set_job_status function definition
SELECT pg_get_functiondef(p.oid)
FROM pg_proc p
JOIN pg_namespace n ON p.pronamespace = n.oid
WHERE n.nspname = 'public' AND p.proname = 'set_job_status';

-- Verify job_component_notes table + RLS
SELECT tablename, rowsecurity FROM pg_tables WHERE tablename = 'job_component_notes';
SELECT policyname, cmd FROM pg_policies WHERE tablename = 'job_component_notes';

-- Count orphan job_components rows (returned 0)
SELECT count(*) FROM job_components jc
LEFT JOIN jobs j ON jc.job_id = j.id
WHERE j.id IS NULL;

-- Count jobs with no estimate AND no serial (returned 0)
SELECT count(*) FROM job_components jc
JOIN jobs j ON jc.job_id = j.id
WHERE j.estimate_number IS NULL AND j.serial_number IS NULL;

-- Count cancelled components (returned 0)
SELECT count(*) FROM job_components WHERE status = 'cancelled';
```
