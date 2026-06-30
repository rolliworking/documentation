# Q-013 — Map watch storage / safe workflow digital state

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-29
**Depends on:** Q-001 (intake), Q-002 (job status), Q-011 (component registration)
**Human answers applied:** A-20260628-009 (carbon work orders paper-only), A-20260628-005 (custody_status foundational), A-20260628-003/D-019 (two-stage intake), D-020 (station_id authoritative), W-37 (drag-and-drop GUI), W-33 (visual asset layer)
**Inputs read:**
- `apps/rs/supabase/migrations/20260110084357_*.sql` — `client_property.custody_status`
- `apps/rs/supabase/migrations/20260110071321_*.sql` — `bins`, `client_property.bin_id`
- `apps/rs/supabase/migrations/20260403213241_*.sql` — `dept_w/b/p/pm` on `client_property`
- `apps/rw/supabase/migrations/20260402224644_*.sql` — `job_components` status/department
- `apps/rw/supabase/migrations/20260515031152_*.sql` — `job_components.station_id`
- `apps/rw/src/pages/ShopFloor.tsx`, `ComponentLookup.tsx`, `StationScanner.tsx`, `WorkQueue.tsx`, `PossessionScan.tsx`
- `apps/rs/src/hooks/useCustodyLog.ts`, `ClientWatchEntryPage.tsx`, `ClientPropertyPage.tsx`
- `documentation/workflows/VIANNA-WORKFLOW.md` §Watch storage, §Handoffs
- `documentation/workflows/MICHAEL-WORKFLOW.md` §6 Final Reflection
- `documentation/discovery/Q-001-receive-watch-module.md`, `Q-002-work-queue-state-machine.md`, `Q-011-component-registration.md`
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**Watch storage & custody** — The physical lifecycle of client watches inside the shop: safe bins (by due date), department bins (polish / band / watchmaker), watchmaker personal bins, night safe retrieval, and release at pickup/ship.

Mike's operational risk: *"Not knowing what's waiting in the safe. No trail or log for chain of custody within the shop."*

This discovery maps **every digital location state** the software tracks versus **what staff track on paper or in memory** — foundational for D-015 (chain of custody), W-33 (visual asset inventory), and W-37 (Shop Floor drag-and-drop GUI).

---

## 2. Trigger

| Event | Physical effect | Digital write today? |
|-------|-----------------|---------------------|
| Package/drop-off receive (Stage 1) | Item into building → safe bin by due date | RS `client_property` + photos; **no safe bin ID** |
| Receive Watch verify (Stage 2) | Confirms possession; dept flags | RS `dept_w/b/p/pm`; **no bin assignment** |
| Vianna sorts to department bins | Polish / band / WM bin | **Human only** — `dept_*` booleans, no bin |
| Label wizard → RW intake | Job enters shop queue | RW `job_components` `in_queue` |
| Watchmaker assignment | Physical handoff to WM bin | RW `assigned_watchmaker` / `assigned_to` |
| Shop Floor drag / scan | Component moves station | RW `station_id`, `status`, `department` |
| `SAFE:main` QR scan | Into safe | RW `in_safe` + `safe_storage` |
| Night EOD | Bins into safe | **Human only** |
| Pickup / ship complete | Released to client | RS `custody_status: released` (ship); pickup gap per Q-006 |
| 3-color carbon work order | Paper travels with components | **Not digitized** (A-009) |

---

## 3. Actors

| Actor | Physical responsibility | Digital surface |
|-------|------------------------|-----------------|
| Vianna (intake) | Safe sort by due date; department bin sort | RS Receive Watch, package receive |
| Manager | Morning bin retrieval from safe | None |
| Watchmakers | Personal bin possession | RW Work Queue, Bulk Assign |
| Polish (Fabian) | Polish room queue setup EOD | RW `refinishing` status only |
| Front desk | Pickup release | RS Pickup Station |
| Shop Floor staff | Station moves, into-safe locks | RW Shop Floor, StationScanner |
| `useCustodyLog` | Package in/out audit | RS only — **not safe contents** |

---

## 4. Inputs

### RS — asset-level (`client_property`)

| Field | Values / type | Location meaning |
|-------|---------------|------------------|
| `custody_status` | `in_custody`, `released` (enum); code also writes `received` | Building custody vs released — **not** safe vs bench |
| `bin_id` | UUID → `bins` table | Parts inventory bins; **rarely set** for watches |
| `dept_w`, `dept_b`, `dept_p`, `dept_pm` | boolean | Routing intent from Receive Watch |
| `is_in_inventory` | boolean | Committed to inventory |
| `notes` | `[received:…]`, `[split_custody:…]` | Unstructured possession hints |

### RW — component-level (`job_components`)

| Field | Values | Location meaning |
|-------|--------|------------------|
| `status` | `in_queue`, `assigned`, `in_progress`, `in_refinishing`, `in_precious_metals`, `in_safe`, `waiting_components`, `fulfilled`, … | Process + coarse location |
| `department` | `watchmaker_bench`, `refinishing`, `band_repair`, `precious_metals`, `safe_storage` | Room routing |
| `station_id` | text (e.g. `lock_after_uncase`, `assign_wm`, `band_tech`) | Shop Floor column — **authoritative target per D-020/W-37** |
| `assigned_to`, `assigned_watchmaker_id` | text/uuid | Tech possession — **not bin number** |
| `assignee_name`, `assignee_initials` | denormalized | Display |

### RW — job-level (`jobs`)

| Field | Purpose |
|-------|---------|
| `assigned_watchmaker` | Job-level WM initials |
| `dept_w/b/p/pm` | Copied from intake |
| `flow` | `STANDARD_FLOW` / `SPLIT_FLOW` / `BAND_ONLY_FLOW` |

### Human / paper only (Vianna workflow)

| Input | Digital equivalent |
|-------|-------------------|
| Safe bin slot (by due date) | **None** |
| Department bin (polish / band / WM) | `dept_*` flags only |
| Watchmaker personal bin | `assigned_to` approximates holder, not bin ID |
| 3-color carbon work order | **None** (A-20260628-009) |
| Polish room queue order (Fabian EOD) | **None** |

---

## 5. Steps (physical vs digital)

### Stage 1 — Into building (D-019 possession)

1. Package or drop-off received → photos to Supabase `attachments` / `package_scan_logs`
2. Vianna places in **safe bin sorted by due date** → **no digital bin**
3. RS may write `custody_status: 'received'` (non-enum drift) or `in_custody`

### Stage 2 — Verification (Receive Watch)

1. Staff confirms components → `dept_*` checkboxes (W/B/P/PM)
2. Per D-020: components **should** be created here — today they are not (Q-011)
3. Vianna sorts to **department bins** physically → digital has booleans only

### Shop floor routing (RW)

1. `rollisuite-intake` creates `job_components` → `in_queue`
2. Bulk Assign / Work Queue → `assign_watchmaker`
3. Shop Floor drag or scan → updates `station_id`, `status`, `department`
4. Lock stations → `in_safe` (no per-bin granularity)
5. `StationScanner` `SAFE:main` → `in_safe` + `safe_storage` (single safe QR)

### Release

1. Ship fulfill → `client_property.custody_status: released`
2. Pickup complete → `pickup_sessions` only today; **missing** custody release (A-015, Q-006)

---

## 6. Outputs

| Layer | What it answers | What it cannot answer |
|-------|-----------------|---------------------|
| RS `client_property` | Is asset in building / released? | Which safe bin? Which WM bin? |
| RS `useCustodyLog` | Packages in/out, pickups, shipments | Safe contents inventory |
| RW `job_components` | Component dept/status/station (when registered) | RS asset link; physical bin IDs |
| RW `ComponentLookup` | Technician + safe_storage card view | Unified client-asset layer (W-33) |
| RW `PossessionScan` | Per-tech possession audit session | Bulk safe audit |
| Paper carbon WO | Dept routing, hand edits | **Out of scope for software** |

---

## 7. Cross-app touches

| Touch | Gap |
|-------|-----|
| RS `client_property` ↔ RW `job_components` | No FK; joined only by estimate/ref inference |
| RS custody log ↔ RW safe state | Independent; "in_safe" in RW ≠ "in_custody" semantics in RS |
| Pickup (Q-006) ↔ RS custody | Exit photos exist; custody release missing |
| Q-011 registration | Without components, Shop Floor cannot represent location |
| D-015 cameras | Future; hooks are `station_id` + audit events |

---

## 8. Edge cases

| Case | Digital behavior |
|------|------------------|
| Split custody (head→safe, bracelet→band) | `[split_custody:…]` in notes only |
| Band done, watch still in WM | `waiting_components` exists; Shop Floor "not functional" per Mike |
| `station_id` NULL (~99.5% historical) | `dotStationFor` heuristics — unacceptable long-term (W-37) |
| `custody_status: received` | Not in DB enum — insert drift |
| Client property `bin_id` | Inventory parts bins, not watch safe |
| Stale safe alert | Work Queue flags `in_safe` >14 days — no bin-level drill-down |

---

## 9. Workarounds observed

| Workaround | Why |
|------------|-----|
| 3-color carbon paper WO | Dept routing when digital flags wrong/missing |
| Manager morning bin ritual | Safe retrieval not in software |
| `dotStationFor` inference | NULL `station_id` backfill |
| `PossessionScan` | Partial substitute for Mike's bulk safe audit wish |
| `ComponentLookup` safe card | Closest "what's in safe" UI — component-level, not asset-level |
| Notes tags `[received:…]` | Unstructured component confirmation |

---

## 10. Open questions

### Q-013-A: Should safe bins be modeled as `station_id` values or a separate `storage_slot` entity?

**Type:** schema
**Why it matters:** W-37 GUI needs drop targets; safe has due-date ordering Vianna uses today.
**Default if no answer in 7 days:** Reuse `station_id` namespace (`safe_bin_A1`, …) with `department: safe_storage`.

### Q-013-B: Is RS `client_property` or RW `job_components` the canonical row for W-33 visual inventory?

**Type:** architectural
**Why it matters:** Two parallel models today; W-33 needs one operator-facing spine.
**Default if no answer in 7 days:** RS asset record + linked component children per D-019/D-020.

---

## 11. Digital state inventory (exhaustive)

### RS tracks

| State | Column / table | Granularity |
|-------|----------------|-------------|
| In building / released | `client_property.custody_status` | Per asset |
| Dept routing intent | `dept_w/b/p/pm` | Per asset |
| Inventory committed | `is_in_inventory` | Per asset |
| Parts bin (optional) | `bin_id` → `bins` | Rare for watches |
| Package/shipment/pickup events | `useCustodyLog` sources | Per event, not inventory |
| Intake photos | `package_scan_logs` | Per arrival |

### RW tracks

| State | Column | Granularity |
|-------|--------|-------------|
| Queue / progress / safe | `job_components.status` | Per component |
| Room | `job_components.department` | Per component |
| Shop Floor column | `job_components.station_id` | Per component (when set) |
| Tech holder | `assigned_to`, `assignee_*` | Per component |
| Job WM | `jobs.assigned_watchmaker` | Per job |
| Transition history | `job_component_logs` | Audit trail (in-app moves) |

### Not tracked digitally

| Physical state | Source |
|----------------|--------|
| Safe bin by due date | VIANNA-WORKFLOW L137 |
| Department physical bins | VIANNA-WORKFLOW L122–127 |
| WM personal bin | VIANNA-WORKFLOW L138–139 |
| Polish room queue order | VIANNA-WORKFLOW (Fabian EOD) |
| 3-color carbon WO | A-009; MICHAEL-WORKFLOW L101 |
| Night safe consolidation | VIANNA-WORKFLOW L139 |

---

## 12. 3-color carbon work orders

Per **A-20260628-009:** Paper forms have **no relationship** to digital asset tracking. They travel with components by department; staff hand-write changes (Mike + Vianna). **Out of scope for D-015** digitization. Rebuild should not attempt to OCR or sync carbon WOs — digital routing must come from Receive Watch component creation (D-020).

---

## 13. Safe audit capability today

| Capability | Exists? | Location |
|------------|---------|----------|
| List all components `in_safe` | Partial | `ComponentLookup.tsx` safe_storage card |
| Bulk scan everything in safe | **No** | Mike's wish — W49/D-015 |
| Per-technician possession audit | Yes | `PossessionScan.tsx` |
| Stale-in-safe alert (>14d) | Yes | `WorkQueue.tsx` |
| RS daily custody log (packages) | Yes | `CustodyLogPage.tsx` — not safe inventory |
| Video traceback | Not wired | W49 constraint |

---

## 14. Minimum invasive gap closures

| Gap | P0 fix | Effort | D-015 / W-37 hook |
|-----|--------|--------|-------------------|
| No component rows | Receive Watch creates components (D-020) | Rebuild | Prerequisite for any location UI |
| NULL `station_id` | Set `lock_pre_queue` at creation; W-37 GUI mutates | Medium | W-37 drop targets |
| RS ↔ RW location split | `shared.asset_id` FK linking `client_property` ↔ `job_components` | Schema | W-33 unified view |
| No safe bin | Add `storage_slot` or `station_id` safe namespace | Small migration | Drag to safe bin in W-37 |
| Pickup custody release | `completePickup` → `custody_status: released` | ~10 lines RS | D-015 release event |
| `custody_status: received` drift | Align enum or use possession table (D-019) | Small | Stage 1 vs 2 clarity |
| Carbon WO routing | **Do not digitize** — fix digital routing at intake | Process + D-020 | — |

---

## 15. Mapping to W-37 and W-33

### W-37 — Shop Floor drag-and-drop GUI

- **Requires:** `station_id` as authoritative field (D-020); components must exist (Q-011/D-020).
- **Must support:** Right-to-left default flow; exceptions (cancel → safe; safe → queue); non-developer staff.
- **Safe bins:** Become draggable targets in `safe_storage` department — today only `SAFE:main` QR exists.
- **Replaces:** `dotStationFor` heuristics when `station_id` NULL.

### W-33 — Visual client asset inventory layer

- **Spine:** RS client asset with linked RW components; show `custody_status` + component `station_id` + holder.
- **Safe view:** Aggregate `department = safe_storage` OR `status = in_safe` with due-date sort from estimate.
- **Search:** By client, ref, serial, status — requires A-004 optional identifiers on components.
- **Not sufficient today:** `ComponentLookup` alone — component-centric, no client-asset rollup.

---

## 16. Comparison to workflow docs

| Source | Finding |
|--------|---------|
| VIANNA-WORKFLOW L122–140 | Physical bin workflow fully described; **0% bin IDs in DB** |
| MICHAEL-WORKFLOW L142–148 | Chain-of-custody risk; bulk safe scan wish |
| Q-011 | No components → Shop Floor location meaningless |
| Q-002 | `in_safe` status exists but lossy RS mapping |
| D-019 | Stage 1 possession photos ≠ Stage 2 verified location |
| D-020 | Receive Watch creates truth; silent failure banned on moves |

---

## 17. Acceptance criteria check

- ✅ Every digital location state named with schema evidence
- ✅ Gaps exhaustive (bins, WM possession, polish queue, paper WO)
- ✅ Carbon WO confirmed paper-only (A-009)
- ✅ Safe audit capability documented (partial only)
- ✅ Concrete minimum-invasive proposals per gap
- ✅ W-37 and W-33 mapping explicit

---

_End of discovery. Q-013 complete._
