---
silo: 9
date: 2026-05-25
status: partial — three architectural fixes shipped, fourth (band pre-fill display for non-Rolex brands) in active debug at chat close
last_validated: 2026-05-25
---

# Silo 9 — Receive-Watch Race Condition + Cross-App Architecture Work

Single long working session. Continuation of silo 8 backlog. Major mid-session tooling shift from Lovable web preview to Cursor IDE + local clones. Three real architectural improvements shipped. Beacon-tracing methodology established.

## 1. Silo Identification

**Number:** 9
**Date range:** 2026-05-24 to 2026-05-25 (single session, ran ~12+ hours)
**Main topic:** Trickle-down data flow across RolliSuite (RS), RolliWorking (RW), and RolliConnect (RC). Specific focus on band-only intake pre-fill, RW receiver field coverage, and RC customer portal timeline display.
**Status at close:** Partially shipped. Three fixes verified on test branch (auto-deployed). One fix (band pre-fill for non-Rolex brands) in active debug via beacon tracing — Cursor proposed Fix 2 + Fix 3 but EST 25530 still rendered Brand pill blank as of last screenshot.

## 2. What Shipped

### Verified live (merged to main via GitHub)

- **Receive-watch race condition fix** — `src/pages/intake/ClientWatchEntryPage.tsx` lines 177-189. Added `hasPrefilledRef` guard to skip the band-only itemType clobber effect when pre-fill just ran for the current estimate. Brand pre-fills correctly for Rolex band-only jobs.
- Test → main merge of all silo 8+9 verified work via GitHub PR (handled by user). Conflict resolved on `.lovable/plan.md` (cleared file).

### Shipped to test branch (auto-deploys)

- **RW receiver Fix A + Fix B** — commit `3bee3db` in `rolliworking/supabase/functions/rollisuite-intake/index.ts`
  - Fix A: `jobs.dept_w/b/p/pm` derived from `resolvedCodes` via `getDeptIntake()` when payload omits explicit booleans; payload booleans still win as operator-intent override
  - Fix B: Each `job_components` INSERT row now sets `department` via `componentDeptFor()` helper — `watchmaker_bench` / `refinishing` / `band_repair` / `precious_metals`
  - Expected to eliminate `BackfillPanel` runs and most `BackfillBandJobs` admin runs

- **RC band-flow Phase 1** — two commits across two repos
  - RS commit `98ce8a36`: `supabase/functions/rc-timeline-status/index.ts` returns sectioned DTO v2 with flow-aware band/split/standard rendering. Adds `version: 2`, `sections[]`, `display.secondary_current_label`. V1 fields preserved at top level for backward compat.
  - RC commit `cf0a56e`: `src/pages/portal/ClientConversation.tsx` ProgressView reads `status.version === 2` + `display.secondary_current_label`, renders second green badge below primary for split-flow jobs.
  - 60s cache TTL on `cache_store`; RC frontend deploys via Lovable pipeline.

### Lovable/Cursor claimed but UNVERIFIED at chat close

- Cursor's Fix 2 (drop `formData.estimateId` from clobber guard) + Fix 3 (sync itemType inside pre-fill effect when isBandOnly true)
- Bracelet-matcher refactor: dropped `/^BR-/i` regex, switched to curated-data check (`department === 'B' && item_type === 'service' && (brand || bracelet_model)`)
- Verification on EST 25530 (beacon) still showed Brand pill blank after Cursor's apply claim. Apply state genuinely unverified — diff existed on disk per Cursor report, but bundle behavior didn't reflect it on localhost test

## 3. What Was Learned (Texture)

### Workflow shift (load-bearing)

Mid-session migrated from Lovable web preview to **Cursor IDE + local clones** at `~/rolliworks/{rollisuite, rolliworking, rollicrm}`. All three repos cloned (test branch each). Vite dev server runs on `localhost:8080`.

- Killed the "stale Lovable preview bundle" debugging loop that ate ~4 hours
- Cursor with 3 repos open = cross-app investigation in one query vs uploading three zips per round
- Diff visible before commit; no proxy/CDN cache between code and verification
- Mike approved Cursor as primary tool for architectural work; Lovable retained for UI scaffolding

### Architectural findings

- **RW shop floor `STATIONS` map (`src/pages/ShopFloor.tsx` L52-113) IS the canonical process flow.** `dotStationFor()` L155-208 derives station placement from `(station_id, component_type, status, department, jobs.status)`. `jobs.flow` is NEVER read by shop floor — placement is purely from existing `job_components` state.
- **RC's `rc-timeline-status` edge function (on RS Supabase) is the source of truth for what customers see.** RC calls its own `timeline-status` proxy → RS's `rc-timeline-status` → returns the timeline DTO.
- **`parts` table is canonical source for brand + bracelet_model on intake pre-fill.** Curated via `/inventory/parts` UI. Pre-fill code at line 849-873 of `ClientWatchEntryPage.tsx` scans `estimate_line_items` joined to `parts` for a curated band service row.
- **`client_property` table has NO `bracelet_model` column.** Schema is `brand`, `model`, `reference_number`, `bracelet_description`. Curated bracelet_model gets folded into `bracelet_description` on receive-watch save.
- **RS push to RW is 18-field caret-delimited string** POST to `https://pkgnrcfqrldwjibghefm.supabase.co/functions/v1/rollisuite-intake` with `Content-Type: text/plain`. No JSON, no auth header. Fired from `src/hooks/useRolliworkingSync.ts:sendIntakeToRolliworking()` lines 99-141 in RS.
- **RW receiver writes 4 of 17 `job_components` columns.** Omits `department`, `assignee_*`, `station_id`, `assigned_*`. This is the structural source of every backfill operation.
- **Three backfill sites compensate for receiver gaps:**
  - `BackfillPanel` inline in `ShopFloor.tsx` L1481-1674 — manual button when components.length===0
  - `backfillWatchmakerJobs` in `ComponentStatus.tsx` L527-579 + silent useEffect mirror L582-623 (runs on every page mount — explains "feels like every time")
  - `BackfillBandJobs.tsx` admin tool with `fixBlankNames` data-repair
- **Receiver L431-434 causes duplicate jobs path** — customer-mismatch on email casing creates parallel job row, leaving `Client #<est>` placeholder orphan with components attached
- **M3KE (knowledge service)** stays separate from RS+RW+RC merger. Fly.io-hosted, own Supabase `ydtyfvquqiiggkqnddxz.supabase.co`, reads RS via PostgREST.

### Reuse-first principle (operational rule going forward)

Every Cursor prompt now leads with: "Before proposing new code, search workspace for existing assets that solve part of this problem." Recognized across silos that fresh-context AI defaults to building new things instead of repurposing what's already in 350-400k LOC across three apps.

### Beacon tracing methodology

Introduced sentinel test data — known values that can be traced through the pipeline:
- Parts row: `BR-Tracking-Beacon` with `brand='Rolex'`, `bracelet_model='Tracking-Beacon-Model'`, `department='B'`, `item_type='service'`
- Estimate 25530 references this part on line 2
- Wherever "Tracking-Beacon-Model" appears = data flowed through; wherever it drops = bug location

## 4. What Was Deferred

### Receiver fix backlog (after Fix A+B verify in production)

- **Fix C — duplicate jobs path** at receiver L431-434 (~15 lines). Match by `estimate_number` + normalized email instead of strict `client_id`. Eliminates `fixBlankNames` backfill flow.
- **Fix D — stop writing `Client #<est>` placeholder** to `jobs.client_name` (~3 lines). Leave null until real name arrives. Eliminates duplicate-job root cause.
- **Fix E — receiver sets `job_components.status` + `assignee_*` on re-intake** when `jobs.assigned_watchmaker` exists (~20 lines). Eliminates silent `ComponentStatus` useEffect backfill.
- **Display bracelet_description on RW `/shop-floor` lookup + work queue** (~5 lines). Helper `formatJobIdentity(brand, model, braceletDescription)` returning "Rolex (Oval Jubilee 20mm)" pattern.
- **Send actual `formData.dateReceived` in RS push** (not hardcoded `today` at line 105 of useRolliworkingSync.ts)
- **Send `formData.notes` in RS push** (currently saved to `client_property.notes` locally only)
- **bracelet_model storage + display on RW** — receiver currently drops with "no DB column" comment line 258. Either add column or use existing `bracelet_description`.
- **Auto-create `inspections` row on intake** linked to job, pre-populated with `bracelet_description` and notes
- **`received_components` drives `job_components` row creation** (riskier — affects component routing)

### From prior silos still open

- Fix 2: Shop floor bulk assign scan bugs (resolver exact-match first, isBandOnly respect)
- Fix 3: QBO push closes components (~10 lines)
- Fix 4: Canonical parser reads E/P/S/D
- Fix 5: Drop redundant ST field + version prefix V:1
- Fix 6: Bracelet description backfill for historical jobs
- J-list: bracket markers structural column, serviceType drop fix, ref-serial splitter helper, bracelet reference catalog (~60 entries) bulk load, Tudor catalog, PDF417 caps raise, drop-off save reliability for EST 25252
- Ship Station inline-notes display (drafted prompt, not shipped)
- Print PART# column bug on some PDFs
- Security advisor went from 4 to 20 findings — pre-existing RLS gaps surfaced by scanner, NOT introduced by tonight's work; triage in daylight

### Architecture rebuild planning (next 2 weeks)

- 2-week planning phase starts now
- 2-3 month rebuild after that
- Decision: RS + RW + RC share one Supabase project; M3KE stays separate
- Canonical Supabase: RS (`djbjwcoddddywkgljuja`) — has reference data, M3KE already points at it, RC's `rc-timeline-status` already lives there
- Open architectural questions (deferred to planning phase):
  - Auth model: one login across apps, or separate logins same DB?
  - Migration approach: big bang nightly cutover, parallel writing, or read-only freeze?
  - Does RC also share merged Supabase, or stay separate via edge functions?
  - Table naming: prefixed (`crm_*`, `repair_*`) vs Postgres schemas vs current names?
  - RLS strategy for shared DB with different operator/owner/customer permissions

### Open question at chat close

Beacon EST 25530 still rendered Brand pill blank after Cursor's claimed Fix 2 + Fix 3. Three possibilities still open:
1. Cursor's diff not actually saved to disk (HMR didn't pick up)
2. Diff saved but bug deeper than caught (Supabase embed shape mismatch — `li.part` object vs `li.parts` array)
3. Race condition we still haven't isolated

Bulldozer prompt drafted (with raw-shape inspection, runtime console.log, intake payload capture) but not sent before chat hit limit.

## 5. What Was Wrong

### Dead ends pursued

- **~4 hours debugging Lovable preview cache.** Tried `.vite` cache wipe, full Vite restart, dist/build deletion, fresh browser, multiple computers, DevTools "Disable cache." Root cause was Lovable's preview proxy serving stale bundle independent of sandbox Vite. Only resolved by abandoning Lovable preview workflow entirely for Cursor + localhost.
- **BR- regex narrowness theory** — Cursor's first proposed fix to bracelet matcher (`/^BR-/i` → curated-data check). Correct refactor in principle but didn't solve the user-facing problem. Beacon with `BR-Tracking-Beacon` part_number passed the regex but pill still rendered blank — proving regex was never the blocker.
- **Pre-fill brand source confusion.** Spent rounds asking "where does brand info live?" across `parts`, `model_references`, `reference_bracelets`, `client_property`. Beacon investigation confirmed pre-fill ONLY reads from `estimate_line_items.part` (the `parts` join). Other tables exist but aren't consumed by pre-fill. Could have skipped that investigation by reading code directly first.

### False summits

- **"It works on EST 24818!"** — early in session. Brand=Rolex pre-filled. Declared trickle-down architecture working. Then EST 25530 (beacon with explicit Rolex brand) rendered blank, exposing that 24818 only worked because brand="Rolex" matched the Brand pill literal (`'Rolex'`), not because pre-fill logic was correct.
- **"Lovable applied the fix"** — multiple rounds. Lovable confirmed diff applied, grep confirmed code on disk, console diagnostic logs never fired in browser. Apply state was genuinely unverifiable from chat alone.
- **"Cursor's Fix 2 + 3 fixed it"** — diff exists in working tree, beacon test on EST 25530 still failed. Either bug deeper than caught or HMR didn't pick up.

### Methodological mistakes

- Repeated DevTools console asks of user. Each round took 5-10 minutes for marginal info. User said "this has become an infinite loop" — fair criticism. Should have asked Cursor (with file/DB access) to investigate sooner instead of routing through user as DevTools relay.
- Treated settled architecture as new design at session start. Mike flagged in real-time. Cost ~2 hours of re-litigating decisions from prior silos. Owned the mistake; should have searched prior silo summaries first.
- Sent user to Supabase SQL Editor multiple times when Cursor couldn't access RLS-protected tables. After 3-4 rounds, user objected and we shifted to having user run queries and paste results, which worked but should have been the pattern from start.

### Anthropic claim quirks

- Lovable's "Applied" message means "diff exists in sandbox file system" — does NOT mean "served bundle includes it" or "user's browser sees it." This distinction was the root cause of the 4-hour cache debugging.
- Multiple times throughout night Lovable confirmed apply but user-visible behavior didn't change. Cursor has different semantics — diff is on disk in user's filesystem, Vite HMR makes it live immediately.

## 6. Key Decisions

### Tooling

- **Migrated to Cursor + localhost** for architectural work (Mike already had Pro account). Lovable retained for UI scaffolding and fast iteration on new pages.
- **Three repos cloned locally** at `~/rolliworks/{rollisuite, rolliworking, rollicrm}` — test branch each. Vite dev on port 8080.
- **Cursor prompt discipline** updated: every prompt now starts with reuse-first search instruction.

### Architecture

- **RS + RW + RC merge to share one Supabase.** M3KE stays separate.
- **RS Supabase as canonical** (`djbjwcoddddywkgljuja`). Rationale: has reference data, M3KE already points at it, RC's `rc-timeline-status` already lives there, RW data is more component-flow and easier to migrate onto RS than vice versa.
- **RC sectioned DTO v2 with backward compat** via `version` field. Phase 1 ships now with minimal RC change (~10 lines for secondary badge). Phase 2 full timeline UI deferred to rebuild.
- **Phase 1 vs Phase 2 split** for RC band-flow: minimum-impact server-side branching ships immediately; richer timeline UI deferred to rebuild scope.

### Rebuild scope

- **2-week planning + 2-3 month build.** Not greenfield rewrite — architecture cleanup of existing 3 apps with shared Supabase.
- **Receiver pipe widening prioritized over UI display work.** Eliminating backfill compensation > prettier displays.

### Methodology

- **Beacon tracing for any cross-app data flow verification.** Known sentinel values traced through pipeline. Single beacon estimate can serve as permanent regression test going forward.
- **Reuse-first prompts.** Every Cursor request starts with "search workspace for existing assets" instruction.

## 7. Artifacts Worth Preserving

### Test fixtures (estimates used as diagnostic anchors)

- **EST 24818** — Rolex Oval Jubilee, part `BR-OVAL-JUB-ALL`, original canonical worked example. Verified pre-fill works.
- **EST 25255** — band-only used for receive-watch testing earlier in session
- **EST 25523** — Rolex Oyster, part `BR-OY-PIN`, regression test case (parts row: brand=Rolex, bracelet_model="Steel Solid link Oyster", department=B)
- **EST 25530** — beacon estimate, part `BR-Tracking-Beacon`, customer "test hui" (mikebhui@icloud.com, 8189254081). Dropped off 2026-05-25 at 10:20 PM. Still rendering blank at chat close.

### Beacon part rows

- `BR-Tracking-Beacon`: brand=Rolex, bracelet_model=Tracking-Beacon-Model, department=B, item_type=service, description="Bracelet Repair - Tracking Beacon"
- `BR-Tracking-Beacon2`: parallel beacon row (brand=Rolex, bracelet_model=Tracking-Beacon-Model2)

### Diagnostic SQL queries

```sql
-- Parts row verification
SELECT id, part_number, brand, bracelet_model, item_type, department
FROM parts
WHERE part_number = 'BR-Tracking-Beacon';

-- Estimate + line_items + parts join (the join receive-watch consumes)
SELECT eli.sort_order, eli.entered_part_number, eli.part_id,
       p.part_number, p.brand, p.bracelet_model,
       p.item_type, p.department
FROM estimates e
JOIN estimate_line_items eli ON eli.estimate_id = e.id
LEFT JOIN parts p ON p.id = eli.part_id
WHERE e.estimate_number = '25530'
ORDER BY eli.sort_order;

-- Receiver fix verification (RW side after intake push)
SELECT j.estimate_number, j.dept_w, j.dept_b, j.dept_p, j.dept_pm,
       jc.component_type, jc.department, jc.status
FROM jobs j
LEFT JOIN job_components jc ON jc.job_id = j.id
WHERE j.estimate_number = '<NEW_EST>'
ORDER BY jc.component_type;
```

### Diagnostic console.logs added (still in place at chat close)

In `~/rolliworks/rollisuite/src/pages/intake/ClientWatchEntryPage.tsx`:
- Line 270: `console.log('ESTIMATE QUERY RESULT', ...)` — inside React Query queryFn
- Line 842: `console.log('BRACELET PART DETAIL', ...)` — after braceletPart IIFE
- Line 877: `console.log('PREFILL DEBUG', ...)` — inside pre-fill effect
- Line 937: `console.log('POST-PREFILL formData', ...)` — useEffect watching formData
- Plus `BR LOOP` per-iteration log inside braceletPart matcher

These should be removed before next merge to main as cleanup.

### Key file paths

- `~/rolliworks/rollisuite/src/pages/intake/ClientWatchEntryPage.tsx` (2690 lines, receive-watch form)
- `~/rolliworks/rollisuite/supabase/functions/rc-timeline-status/index.ts` (RC timeline source of truth)
- `~/rolliworks/rolliworking/supabase/functions/rollisuite-intake/index.ts` (643 LOC, RS→RW receiver)
- `~/rolliworks/rolliworking/src/pages/ShopFloor.tsx` (1675 LOC, primary operator view)
- `~/rolliworks/rollicrm/supabase/functions/timeline-status/index.ts` (RC proxy to RS)
- `~/rolliworks/rollicrm/src/pages/portal/ClientConversation.tsx` (ProgressView component)

### Repo URLs

- RS: `https://github.com/rolliworking/rollisuite-ed468227`
- RW: `https://github.com/rolliworking/rolliworking-42f760d5`
- RC: `https://github.com/rolliworking/rollicrm-scaffold`
- Branch: `test` (Lovable connected, publishes to `rolliworks.com` etc.)

### Lovable project IDs

- RS: `939ed0e0-6220-40e3-a195-65d7c466acfe`
- RW: `840c6e92-a516-426f-853f-fd7040707d11`
- RC: `fd00c167-a7e5-47b8-9e32-483593e792ef`

### Supabase project IDs

- RS: `djbjwcoddddywkgljuja` (canonical for merger)
- RW: `pkgnrcfqrldwjibghefm`
- M3KE: `ydtyfvquqiiggkqnddxz` (stays separate)
