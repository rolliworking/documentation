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


---
silo: 8
date: 2026-05-15
intent: APPEND this content to existing handoff/current.md in the repo
last_validated: 2026-05-15
---

# Silo 8 contributions

The following sections update / extend the prior handoff content. They are written to be appended (not replaced) so the doc history accumulates.

## New workflow conventions

### Apply messages must restate surgical framing

Bare "approved" leaves room for Lovable to interpret broadly â€” pull in nearby cleanups, add "improvements," append follow-up suggestions. Restating the framing at apply time keeps blast radius constrained.

- Original prompt: `# Mode: Surgical fix â€” plan first, apply after I confirm`
- Apply trigger: `# Mode: Surgical fix â€” apply the plan exactly as shown` + restated constraints

Example apply message structure:
```
# Mode: Surgical fix â€” apply the plan exactly as shown

Apply [feature] plan from your previous message. Constraints carry over:

- Touch [N] file(s): [list]
- No new dependencies
- No schema changes
- No "while we're here" cleanups
- No follow-up suggestions appended
- Use the existing [pattern/component]
- Apply exactly the diffs shown â€” no additions

After applying, confirm:
- Exact lines changed (file + line numbers)
- No other files touched
- No new imports beyond what the plan specified
```

### Don't proactively suggest stopping

Mike pushed back on this twice in silo 8. The "you're tired, save it for morning" framing is annoying, condescending, reduces trust. Trust the user to manage their own time. Only mention rest if the user raises it first.

If a session is visibly degrading on the user's side (paste mix-ups, contradictory instructions), Claude can mention it ONCE. Then drop it.

### Don't anchor to a time without checking

Claude anchored to "11:43pm hard stop at 2am" mid-session and kept referencing those times into Mike's afternoon (4:08pm est). Sessions can pause and resume. Don't assume continuity of time. If timing matters, ask.

### When Lovable says "fully patched," ask for ALL writers

Silo 7's B1+B2 verification was technically correct on the surface it covered. Silo 8 follow-up found 12+ other writers that silo 7 didn't audit. Going forward, when Lovable claims a shared-code surface is "fully patched":

> List EVERY writer to this table/column in the entire codebase, not just the obvious ones. Include async handlers, dialog flows, edge functions, and any other place that touches the same column.

This is the audit prompt that surfaced the silo 8 finding.

### Three-AI workflow is load-bearing

A week of productive shipping has confirmed it. Lovable on the live codebase with DB access for verification, Claude on planning + cross-checking confident claims, Mike as deliberate bus. Neither tool alone would have caught what got shipped this week.

The discipline drivers are:
- "Surgical fix â€” plan first" framing
- Research prompts before fix prompts
- Apply-message-restates-framing
- "List every writer" follow-ups

## New vocabulary

### Push vs Shove

Two distinct Ship Station actions on the SO detail page:

- **Push to Ship Station** â€” Existing action. Local DB UPDATE setting `fulfillment_channel='ship'` (plus destructive downgrade branch when status='shipped'). Does NOT bypass `isPaid` gate. Not a real external integration.
- **Shove to Ship Station** â€” Designed silo 8, not yet built. Admin-only super override. Sets `is_paid=true, paid_at=now(), fulfillment_channel='ship'`. Audits to `payment_bypass_log` with reason. SMS via `send-security-alert`.

Operational rule: a push routes, a shove forces.

## New architectural findings

### station_id is only populated by 2 of 14+ writers

ShopFloor.tsx's `commitDrop` (line 664) and `handleCommit` (line 1033) are the only writers populating `job_components.station_id`. The other 12 writers (smart-create, NewJob, BandKanban, ComponentStatus, BackfillBandJobs, StationScanner, component-creation.ts, rollisuite-intake, rollisuite-change-order, rollisuite-so-fulfilled) do not.

Result: 221/222 components have NULL station_id; 130/136 logs do (42 of which are `backfill` rows where NULL is correct).

**Implication:** any UI surface reading station_id must treat NULL as dominant. Department fallback is the primary display path. The Job History Lookup pills shipped silo 8 deliberately don't read station_id.

See `known-gaps/station-id-only-2-of-14-writers.md` for the full writer audit.

### Role schemas differ between RW and RS

- **RW**: `owner` / `manager` / `staff`. No `admin` role exists.
- **RS**: `admin` / `manager` / `office` / `front_desk` / `staff` / `band_room` (AppRole enum at `src/types/database.ts:13`). No `owner` role exists.

In RS, `admin` IS the super-user / owner-equivalent. Gate super-user features on `useAuth().isAdmin`. When Mike says "owner-only" in an RS context, it means admin.

### Ship Station is NOT a real external integration

"Ship Station" refers to the internal `/sales/ship-station` queue page (`PaidUnshippedOrdersList.tsx`). "Push to Ship Station" on the SO detail page (lines 2069-2093) is a local DB UPDATE only. There is no ShipStation API integration.

Push side effects worth knowing:
- Always sets `fulfillment_channel = 'ship'`
- On downgrade from `status='shipped'`: clears `tracking_number`, `ship_date`, zeroes all `so_lines.shipped_qty` â€” **destructive, no confirmation dialog**

### `set_job_status` cannot be reused from triggers

The RPC has `auth.uid()` checks that fail for service-role writers (edge functions, intake). Plan D's trigger reimplements the write + RS push directly with SECURITY DEFINER.

### jobs.status is single-dimensional â€” limits RS visibility

A job has multiple components. Component statuses can diverge. `jobs.status` is one value. So RS, which reads `jobs.status` only, cannot show "bracelet done, watch still in progress." See `known-gaps/jobs-status-single-dimensional-no-partial-completion.md`.

### QBO sync has a small window for paid invoices to fall through

Cron checks 21 oldest open invoices per run. Pre-silo-8 manual button only re-checked last 7 days. SOs in the gap (older than ~3 weeks, paid in QBO after the cron's window) stay locally-unpaid until someone notices. Part 2 widening (shipped silo 8) catches them on manual click.

## Shipped silo 8

**RW production:**
- Job History Lookup component status pills â€” per-component pills below existing 3 header pills, reusing `Badge` component, fallback `assignee_initials` â†’ `department` â†’ status alone.

**RS production:**
- SO 500950 unblock Part 1 (data UPDATE: `is_paid=true, paid_at=now()` on row `a5e827c2-1eb9-4094-b3ba-c79d020d3667`).
- Sync QBO button widening Part 2 (`PaidUnshippedOrdersList.tsx` lines 350-358, 414; "any unpaid row with qboInvoiceId, oldest-first, cap 25").

## Designed but NOT YET applied

- **Plan D â€” bump_job_to_in_progress trigger.** Verified pg_net, no existing triggers, auth.uid reimplementation justified. Backfill of 4 candidates skipped. Surgical-fix prompt not yet drafted.
- **Plan E â€” Shove to Ship Station.** Admin-only override. Cloned-from-PaymentBypassDialog modal. Reuses `payment_bypass_log` with `station='ship_station'`. Surgical-fix prompt drafted but not yet pasted.
- **Fix B â€” handleQuickBarcodeSubmit re-entry guard.** One line in `SalesOrderDetailPage.tsx`. Touched silo 7 and silo 8; neither landed. Silo 8 sent prompt to wrong AI (RW Lovable correctly refused).
- **Deep-link from JobHistoryLookup to Shop Floor.** Researched, ~6-10 lines in ShopFloor.tsx. Lower priority after pills shipped.

## New urgent backlog (silo 8)

- **SO 501718 QBO Customer Memo length error** â€” push fails with ValidationFault, 1016 > 1000 chars. Source of overage unknown. Customer-blocking.
- **QBO webhook CloudEvents deadline** â€” May 15, 2026 (today). Whether RS migrated unknown. If not, payment-status webhooks fail silently.
- **2 candidate stuck SOs** â€” 55153, 55309. Pending Part 2 button verification.
- **Push to Ship Station destructive downgrade** â€” wipes shipment evidence with no confirmation when status=shipped.
- **Premature `status='fulfilled'` source bug** â€” Part 3 from SO 500950 investigation.

## See also (from this silo's bundle)

- `silos/silo-8.md` â€” full silo writeup
- `known-gaps/station-id-only-2-of-14-writers.md`
- `known-gaps/qbo-customer-memo-1000-char-limit.md`
- `known-gaps/qbo-webhook-cloudevents-migration.md`
- `known-gaps/push-to-ship-station-destructive-downgrade.md`
- `known-gaps/jobs-status-single-dimensional-no-partial-completion.md`
- `decisions/2026-05-15-pills-over-current-state-block.md`
- `decisions/2026-05-15-shove-to-ship-station-design.md`
- `decisions/2026-05-15-plan-d-trigger-approach.md`
- `decisions/2026-05-15-so-500950-three-part-split.md`
- `fixtures/so-500950.md`
- `fixtures/so-501718.md`
- `fixtures/est-24392-larissa-oprysk.md`
- `fixtures/plan-d-backfill-4-candidates.md`
- `fixtures/so-55153-and-55309-stuck-candidates.md`


---
appended_by_silo: 9
appended_date: 2026-05-25
---

## Silo 9 contributions (appended 2026-05-25)

### Current state at silo 9 close

**Three shipped fixes, one in active debug:**

1. âœ… **Receive-watch race condition** â€” merged to main. Brand=Rolex pre-fills correctly on band-only jobs via `hasPrefilledRef` guard at `ClientWatchEntryPage.tsx` L177-189.

2. âœ… **RW receiver Fix A+B** â€” shipped to test branch as commit `3bee3db`. `dept_*` derived from service codes; `job_components.department` set at INSERT.

3. âœ… **RC band-flow Phase 1** â€” shipped as RS `98ce8a36` + RC `cf0a56e`. Sectioned DTO v2, version-aware backward compat, secondary badge for split-flow.

4. â³ **Beacon EST 25530 pre-fill** â€” Cursor applied Fix 2 + Fix 3 (clobber guard + itemType sync) but Brand pill still blank. Bulldozer prompt drafted, not sent. Open at chat close.

### What the next silo should pick up

**Immediate:**
- Send bulldozer prompt for beacon EST 25530 (in known-gaps/beacon-est-25530-blank-prefill.md)
- Verify Fix A+B by saving a new band-only intake from RS, then checking RW shop floor for the new job (no BackfillPanel needed, dept_b auto-set)
- Verify RC band-flow on a band-only customer portal URL â€” should show 6-step linear timeline

**Receiver fix backlog (after verification):**
- Fix C: duplicate jobs path (~15 lines)
- Fix D: stop writing "Client #<est>" placeholder (~3 lines)
- Fix E: receiver sets job_components assignee_* on re-intake (~20 lines)
- Display bracelet_description on RW shop floor + work queue (~5 lines)
- Send real dateReceived in push (not today)
- Send notes in push

**Rebuild planning (2-week phase starts now):**
- Six architectural decisions to document (auth, migration, RC inclusion, table naming, RLS, schema organization)
- See `decisions/2026-05-25-shared-supabase.md` for open questions list

### Workflow change effective immediately

- **Cursor + localhost** is now the primary tool for architectural work
- Three repos cloned at `~/rolliworks/{rollisuite, rolliworking, rollicrm}` (test branch each)
- Lovable retained for UI scaffolding only
- All future Cursor prompts MUST start with reuse-first search instruction
- Beacon fixtures (`BR-Tracking-Beacon` + EST 25530) are now baseline for cross-app verification

### Open architectural questions for planning phase

- Auth model: one login across RS+RW+RC, or separate per app?
- Migration approach: big bang, parallel writing, read-only freeze?
- Does RC also share merged Supabase, or stay separate via edge functions?
- Table naming: prefixed (crm_*, repair_*) vs Postgres schemas vs current names?
- RLS strategy for shared DB with operator/owner/customer differentiation

### Lessons carried forward

- Don't trust "applied" claims without verification on actual served bundle
- DevTools console relay through user is slow and error-prone â€” prefer giving Cursor direct file/DB access
- Race conditions in React effects are subtle â€” beacon with explicit known values exposes them
- The receive-watch page is the canonical pain point for trickle-down architecture â€” it's where 4 silos worth of bugs live
- Same string can mean different things in different parts of codebase (`waiting_components` = "Awaiting reunion" component-level AND "Awaiting Invoice" job-level) â€” naming collisions caught in silo 9 audit

### Test fixtures established

- EST 24818 â€” Rolex Oval Jubilee, BR-OVAL-JUB-ALL, working reference
- EST 25523 â€” Rolex Oyster, BR-OY-PIN, regression case
- EST 25530 â€” beacon, BR-Tracking-Beacon, traces full pipeline
- BR-Tracking-Beacon and BR-Tracking-Beacon2 parts rows in /inventory/parts

These should survive any cleanup. They are intentional regression-test data.


---
last_updated: 2026-05-27
last_silo: 10
last_silo_status: shipped (root fix verified); RSâ†’RW research complete
next_silo_focus: TBD â€” depends on operational evidence for markerless flow
---

# RolliWorks Project â€” Live Handoff Doc

**Read this first in every new chat.** This is the running synthesis of what's known about the project across silos.

## What this project is

RolliWorks ecosystem â€” three apps:
- **RolliSuite (RS)** â€” operator-facing ERP. Estimates, receive-watch, intake labels, package receive, customer management. Lives at `~/rolliworks/rollisuite`.
- **RolliWorking (RW)** â€” shop-floor app. Job tracking, technician work queues, component routing. Lives at `~/rolliworks/rolliworking`.
- **RolliCRM (RC)** â€” customer-facing portal. Order timeline, inspection approvals. Lives at `~/rolliworks/rollicrm`.

All three on Supabase backend. Frontend React/TypeScript. Owner: Mike Hui, solo dev with Claude + Cursor + Lovable assistance.

## What's stable / shipped

- **Receive-watch prefill works** for both URL-param entry (`?estimate=N`) and scan-input entry. Silo 10 commit `f2e1c0ba` on `rollisuite` `test` branch (pending merge to main after regression).
- **BR-Tracking-Beacon fixture** lives permanently in RS parts table. Use for diagnostic tracing â€” see `fixtures/br-tracking-beacon.md`.
- **Diagnostic logs** live in `src/pages/intake/ClientWatchEntryPage.tsx`: `[PREFILL EFFECT ENTRY]`, `[BEACON TRACE]`, `[MATCHER CHECK]`, `BR LOOP`, `BRACELET PART DETAIL`, `POST-PREFILL formData`, `LINE ITEMS RAW`, `ESTIMATE QUERY RESULT`. Don't strip blindly.

## Working methodologies (USE THESE)

- **Beacon strategy** â€” sentinel-value test fixtures for contamination-free debugging. `decisions/2026-05-25-beacon-strategy.md`
- **Bulldozer methodology** â€” research entire pipeline in one pass before any fixes. `decisions/2026-05-25-bulldozer-methodology.md`
- **Ground-truth verification** â€” never speculate about DB content. Run a query, paste the raw result. If you can't, say so.
- **Two independent research passes** â€” for cross-app debugging, get RS-side and RW-side researched separately and cross-validate.

## Anti-patterns (AVOID THESE)

- **False summits.** Don't announce "found it" / "got it" until the bulldozer table is complete and the proposed fix accounts for every gap. User has counted 50-100+ premature claims across silos 7-10. They cost trust.
- **Inference as fact.** Don't extrapolate from one signal to a system-wide claim. If a researcher says "to test path X you'd need fixture Y," that doesn't mean fixture-Y-shaped data exists in production.
- **Fix before research on cross-layer bugs.** Apply bulldozer at hour 5-10 of any bug spanning multiple apps or stages, not at hour 95.

## Open known gaps (status: deferred at silo 10 close)

| Gap | Severity | Status |
|-----|----------|--------|
| Markerless estimates â†’ BAND_ONLY_FLOW misclassification | Code blast-radius large; operational impact unknown | Deferred pending evidence |
| 9 setFormData paths in ClientWatchEntryPage.tsx | Architectural debt | Defer refactor until stability proven |
| RW data blind spots (bracelet_description, received_components not rendered) | Operator trust ("data going into trash") | Open, requires multi-track work |
| Package-receive trickle failure | EST 24833 example | Open â€” pre-selection bug + reconciliation rule both missing |
| Prefix-mismatch dead matchers | Mostly cosmetic | Coupled to markerless decision |
| RW caret payload positions 8, 10, 11 dead | Bulldozer bugs 7, 11 | Open â€” fix during data contract redesign |

Full detail in `known-gaps/` directory.

## Repo locations

- RS: `~/rolliworks/rollisuite`
- RW: `~/rolliworks/rolliworking`
- RC: `~/rolliworks/rollicrm`
- Documentation: `~/rolliworks/documentation` (this repo)

## Branch conventions

- Work happens on `test` branch
- Merge to `main` only after regression verification
- Lovable picks up changes from GitHub
- Cursor + localhost is the dev environment

## Where to look first for any new task

1. **Read this `handoff/current.md`** (you are here)
2. **Read the most recent `silos/silo-N.md`** for what just happened
3. **Read relevant `known-gaps/*.md`** if the task touches a known problem area
4. **Read relevant `decisions/*.md`** to understand why things are the way they are
5. Only then dive into code

## Silo log

| Silo | Topic | Status |
|------|-------|--------|
| 1-9 | (Pre-documentation-repo; texture not preserved) | shipped or deferred â€” see commit history |
| 10 | Receive-watch prefill root cause + RSâ†’RW research | shipped (root fix); research complete |

## How this doc gets updated

At each silo close, append a new "Silo N close" section below summarizing:
- What changed in known gaps (closed? new? severity shifted?)
- What new decisions were made
- What's the next silo's likely focus

Don't rewrite this doc â€” append. Newest at the bottom. Old text stays as historical context.

---

## Silo 10 close â€” 2026-05-27

### Changed
- New decision: documentation repo established (this).
- Known gaps documented (5 files): markerless-estimates, prefix-mismatch, rw-data-blind-spots, nine-setformdata-paths, package-receive-trickle.
- Fixture documented: br-tracking-beacon (permanent).
- Methodology decisions documented: beacon strategy, bulldozer methodology, defer markerless flow fix.

### Status of silo 10 work
- Receive-watch prefill SHIPPED via `f2e1c0ba` (URL-param `.or()` normalization) + uncommitted scan-path URL handoff. Both verified working on EST 25530.
- Regression test on EST 25523 and 24818 PENDING before merge to main.
- Three pre-root commits (`a3e4a339`, `70505bdf`, `cb322d0d`) shipped real but secondary fixes.

### Next silo likely focus
Depends on the three operational evidence questions:
1. Has any operator reported watch jobs flipping to band-only on receive-watch?
2. Has any customer complained about wrong portal timeline?
3. Do shop floor techs rely on component dots, or route work informally?

If any "yes" â†’ silo 11 = markerless flow fix.
If all "no" â†’ silo 11 = RW shop floor rendering of bracelet_description (smaller scope, addresses "data going into trash" complaint).


---

## RC session close — 2026-05-27

Full detail: [rc/handoffs/session-handoff-2026-05-27.md](../rc/handoffs/session-handoff-2026-05-27.md)

### Production incident (recovered)
- Internal notes deploy (`05121ba`) shipped frontend before migration → `is_internal` column missing → silent send failure + inbox color loss.
- Migration run manually in Supabase SQL editor → recovered.

### Urgent follow-ups
- **Abandon @mention internal notes** — Vienna `@mike` leaked to customer email; disable or revert feature; keep Mark Unread bump + `is_internal` column/RLS.
- **Reply token UI** — `<reply_token>@reply.rolliworks.com` block missing from ConversationPanel after merge; re-add.

### Shipped / verified today (RC)
- Shared team read state, inbox case-insensitive team_users lookup, Vienna manager Send Portal Invite fix, per-customer reply tokens (271 backfilled), Mark Unread bump via `last_message_at`, `get-rc-portal-link` endpoint verified.

### Ingested
- RC documentation bundle → `documentation/rc/` (architecture, features, plans, prompts, security, technical-debt, handoffs).

---

## Silo 11 close - vendor pricing split

### New known gap

- **RS vendor pricing has two conflicting tables.** `vendor_parts` has 100 rows and is the only source read by `generate_reorder_suggestions`; `part_vendor_prices` has 1,539 rows from CSV pricing import and is invisible to reorder. The 7 overlapping `(part_id, vendor_id)` pairs all disagree by more than $0.01, so a mechanical sync is unsafe without operator judgment. Full detail and diagnostic SQL: `known-gaps/vendor-pricing-dual-table-split.md`.

### Product implication

The silo 11 reorder x vendor-comparison work can union both tables for that new surface and surface conflicts, but that does not close the underlying gap. AutoPO, PartsVendorsTab, and `generate_reorder_suggestions` still need a single-source-of-truth decision or a read-side union strategy.
