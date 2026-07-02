# Wishlist — Rolliworks App Suite Backlog

> **What this is:** The captured backlog of requested features across the suite. It serves two
> jobs at once:
> 1. **Park** items so they don't pull focus from current priorities (see `NORTH-STAR.md` §5).
> 2. **Constrain** today's design — a few items must be *accounted for now* (data model,
>    seams) even though the feature is built later.
>
> **Status:** Living document. Add items, retag, mark done. Newest items at the bottom of each
> section unless reprioritized.
>
> **How to read the tags:**
> - `[BACKLOG]` — build later. Park it. Don't design around it unless it's cheap to.
> - `[CONSTRAINT]` — design for it *now*. The data model / seams must leave room for this even
>   though the feature itself is built later. These are the ones that shape current work.
> - `[IN-SCOPE]` — part of a current priority; being worked now.
> - `[DONE]` — shipped/verified. Keep for the record.
>
> **App key:** RC = RolliConnect · RS = RolliSuite · RW = RolliWorking · RT = RolliTime ·
> AUTH = Authenticator app (future) · QBO = QuickBooks Online · M3KE / Jarvis = future modules.
>
> **ID convention:** New wishlist entries use hyphenated IDs (`W-##`), matching `D-###` and
> `Q-###` in the decisions and questions registries. Legacy table rows from the Workflow Q&A
> import use compact form (`W29`–`W49`); all new detailed entries use `W-##`.
>
> **Source:** Workflow Q&A additions (2026-06-25) merged below as W29–W49. See
> `workflows/MICHAEL-WORKFLOW.md` and `workflows/VIANNA-WORKFLOW.md` for operational context.

---

## How to use this with the pipeline

- When pointing Qwen at an app, have it read this file's relevant section so it documents the
  app already knowing which future items must be accounted for.
- Every `[CONSTRAINT]` item is a note to carry into the rebuild brief for the app it touches —
  "leave a seam for X."
- When triaging what to build next, work from `NORTH-STAR.md` §5 first; this list is the
  parking lot, not the priority queue.

---

## Priority signal (Workflow Q&A 2026-06-25)

| Tier | Theme | Wishlist IDs |
|------|-------|--------------|
| 1 | Chain of custody / theft prevention | W49, W7 |
| 2 | High-leverage workflow gaps | W29–W36 |
| 3 | Watchmaker bench improvements | W26, W27, W37, W-37 |
| 4 | Performance + accountability | W39, W40, W-39 |
| 5 | Pickup + handoff polish | W41, W42, W-38 |
| 6 | Communication + data polish | W12, W16, W43–W48 |

---

## RolliConnect (RC)

| ID | Status | Item |
|----|--------|------|
| W14 | `[IN-SCOPE]` | **Process flow charts for returning clients.** Turn off old process flow charts when clients return; support showing multiple flows for multiple ongoing/old jobs. *(Tied to the current client-portal process-flow priority — handling non-linear states like in-testing/uncased and multiple flows.)* |
| W16 | `[IN-SCOPE]` | **Message routing & request merging.** Improve message routing; add ability to merge requests; resolve one-client-multiple-threads fragmentation. **Also:** client merge/split + labels — RC currently creates duplicate clients; conversations get fragmented. *(Current priority — the message-thread fracturing fix.)* |
| W10 | `[CONSTRAINT]` | **Estimate reply captured to RC.** When a client replies to an estimate, add the conversation to RC. *(Touches the threading model — design the conversation data so estimate replies attach to the right unified thread.)* |
| W15 | `[BACKLOG]` | **Date requests on RC.** Have the request dated. |
| W17 | `[CONSTRAINT]` | **Magic link to RC + client portal connection.** Add a magic link to RC on the inspection-notes HTML page; connect questions/comments to the client portal on their page. *(Touches portal + threading — design seams so portal questions land in the unified conversation.)* |
| W44 | `[CONSTRAINT]` | **Email CC/BCC into RC threads.** Replies to client emails land in the RC conversation thread; avoids fragmentation between Outlook and RC. *(Workflow Q&A Tier 6.)* |
| W47 | `[BACKLOG]` | **"Bump email" from a different domain.** For when client emails go to junk; catches lost replies. *(Workflow Q&A Tier 6.)* |

---

## RolliSuite (RS)

| ID | Status | Item |
|----|--------|------|
| W6 | `[BACKLOG]` | **Improve shop work orders.** Mark work orders complete; export to QBO. |
| W18 | `[BACKLOG]` | **Shop work orders: QBO flow, payment matching, outbound notes.** Invoice-like flow when pushing to QBO; match work-order payments to the bill; leave a note on the estimate page (custom notes) when an item is sent out; scan a label (Est# or customer watch label as Part#) and leave a comment. *(See also W45 for automated document attachments.)* |
| W19 | `[BACKLOG]` | **Auto-populate client watch ref/serial onto estimate.** Add ref-serial to the client estimate automatically after the receive page is done; eliminate manual entry per job. |
| W20 | `[BACKLOG]` | **Email vendor a shop work order before sending item out.** Use an email template + mailto: function. |
| W21 | `[CONSTRAINT]` | **Daily hit list / to-do lists on RS.** Radio buttons for a curated per-person view + an "All" button for admin; to-do items; add items to someone else's list (e.g. `#vienna order paper`); future tie-in to Jarvis. Notes from mid-process job changes should land here. *(See also W31 for 7-day stagnation.)* |
| W22 | `[BACKLOG]` | **Better warranty flow** for the watchmaker room and small jobs. *(See also W32 concierge workflow for rush/small/outsourced jobs.)* |
| W23 | `[BACKLOG]` | **Fix estimate Part# column.** Estimates currently print the description in the Part# column — fix this. |
| W24 | `[CONSTRAINT]` | **Organize trade-account photos by Est#.** Trade accounts accumulate hundreds of photos; organize by estimate number. *(See also W48 for date-based galleria folders — some clients have 30–50 watches × 10–40 photos each.)* |
| W31 | `[CONSTRAINT]` | **7-day stagnation tracker.** Jobs received but not progressed within 7 days of receipt; surface in overview before Work Queue entry; note field + reset-7-day-timer. Backstop for staff accountability. *(Workflow Q&A Tier 2.)* |
| W34 | `[CONSTRAINT]` | **Multi-estimate per shipping label at intake.** Customer ships own + friend's job on one label; assign incoming package to multiple estimate numbers. *(Workflow Q&A Tier 2.)* |

### AI-assisted intake verification — scaffolding now, AI later (W-36)
**Source:** Session 2026-06-28 (follow-up to D-019 two-stage intake pattern)
**Description:** Build the structural scaffolding for AI-assisted comparison of intake photos against staff-reported received components. AI reliability is not yet at theft-prevention grade (false positives too costly), so the AI step is deferred — but the scaffolding around it is built now so future activation is a switch-flip, not a rebuild.

**What gets built now (scaffolding):**

1. **Photo capture discipline**
   - Standard angles enforced at Stage 1: dial view, caseback view, profile/side view, components-separated view (when multi-item)
   - Every photo linked to: ref-serial, estimate ID, intake stage (possession vs verification), photographer, station ID, timestamp
   - High-resolution storage (no compression on Stage 1 photos — they may train future models)
   - Photo metadata exposed via shared.intake_photos with all linking fields above

2. **Schema slot for AI-detected items**
   - `intake_photos.ai_detected_items` (JSONB, nullable) — populated when AI is wired in; empty until then
   - `intake_photos.ai_confidence` (numeric, nullable) — same pattern
   - `intake_photos.ai_model_version` (text, nullable) — for tracking which model produced which detection
   - `intake_photos.ai_run_at` (timestamptz, nullable) — when AI processed this photo
   - All nullable, no current code depends on them being populated

3. **UI hook in the verification gate (Stage 2)**
   - Receive Watch page has a placeholder area for "AI-detected items" warning banner
   - When ai_detected_items is null (current state), banner doesn't render
   - When ai_detected_items is populated (future state), banner compares against staff's received_items list and highlights mismatches
   - Toggle in Setup/Admin to enable/disable globally

4. **Audit-log integration**
   - When the AI banner fires (future state), every staff override or acknowledgment logs to shared.audit_log per D-019
   - Pattern of overrides over time becomes a queryable signal (no accusation, just data)

**What's deferred (AI activation):**

- The actual call to a vision model (Gemini Flash, Claude Haiku, or specialized authenticator-grade model)
- The comparison logic between AI output and staff-reported items
- Confidence thresholds for when to show the banner
- The training-data pipeline that improves the model over time

**Activation triggers (when to wire AI in):**

- Vision models reliably distinguish "watch head only" vs "complete watch" without false positives at >95% accuracy on Rolliworks intake photos
- Or the Authenticator app (Year 1 build plan) matures component detection to that bar via internal training
- Or operator decides the Tier 1 soft-warning version (just flagging "review needed" on category mismatches) is good enough — that's achievable today and could ship sooner if priorities shift

**Why scaffold now, even though AI isn't ready:**

- Photo discipline established at Stage 1 immediately compounds in value — the photo library accumulates as training data
- Schema migrations later are more expensive than schema slots reserved now
- The verification gate UI surface is being built anyway per D-019; adding the placeholder area costs ~10 minutes more than not
- D-015 (chain of custody) and D-019 (two-stage intake) both benefit from photo discipline regardless of AI

**Priority:** Medium — scaffolding work folds into D-019 build; AI activation is its own future decision

**Dependencies:**
- D-019 (two-stage intake pattern) — defines where the scaffolding lives
- D-015 (chain of custody / theft prevention) — defines why this matters
- RolliTime dial photo library — first major consumer of photo discipline (other than W-36 itself)
- Future Authenticator app — likely supplier of the visual-reasoning model

**Status:** Wishlist (scaffolded with D-019 build; AI activation pending tech maturity)

**Related:**
- W-33 (visual asset inventory layer) — UI surface where AI mismatches would surface
- W-34 (intake-to-inventory audit log) — where AI/human disagreements get logged

### Shop Floor drag-and-drop GUI (W-37)
**Source:** A-20260628-007 (Q-004-2)
**Description:** Build a drag-and-drop GUI for Shop Floor station management. Staff must be able to manually move components between stations to handle exceptions: client cancels/aborts (move to safe), client changes mind (move from safe back into queue). Flow is typically right-to-left, but the exceptions are real and recurring. The GUI must be usable by non-developer staff — they understand visual interfaces, not code or status menus.
**Priority:** High (Shop Floor cannot function reliably without manual override capability)
**Dependencies:** Rebuild's station_id authoritative field (D-020), component data model

### Side-by-side intake/return photo comparison at pickup (W-38)
**Source:** A-20260628-014 (Q-006-A)
**Description:** At pickup time, display intake photos (drop-off or shipping receive — both as "before") alongside the items being released at pickup ("after"). Operator visually confirms the right items are being returned. Mislabeling errors during return do happen and this comparison would prevent them.
**Priority:** High (theft and mislabeling risk; foundational to D-015 chain of custody)
**Dependencies:** D-019 (two-stage intake with photos as Stage 1 evidence), unified intake photo source

### Scale planning to 100k+ customers (W-39)
**Source:** A-20260628-010 (Q-005-A addendum from Michael)
**Description:** The rebuild's data model and sync paths must be designed for 100k+ customers, not just the current ~10k. This affects: customer sync chunking strategy, indexing, RLS policy performance, search/filter UX, and any operation that touches the full customer set (reports, audits, exports). Cron-driven full-scan operations must be replaced with chunked, idempotent, monitored equivalents.
**Priority:** Medium (not blocking but informs every schema and sync design decision)
**Dependencies:** Canonical data model design, sync architecture (D-020), customer master plan

### AI-generated warranty summary reports (W-40)
**Source:** A-20260629-003 (Q-008-A)
**Description:** When a warranty issue arises or a client questions the quality of work performed, an AI-generated summary report can be produced pulling together: RolliTime test data (before/after tolerances, accuracy readings, power reserve), inspection photos (from D-021 index), service history (parts replaced, work performed, technician notes), and job timeline. The report is client-facing — a professional summary of what was done and the technical evidence supporting the quality of the work.

**Use cases:**
- Warranty claim response: "Here's what we did, here's the test data proving it was in spec at delivery"
- Client quality inquiry: "Here's the full record of your watch's service, with photos and measurements"
- Insurance documentation
- Post-service confidence-building (proactive summary sent with pickup notification)

**Priority:** Medium-high (strategic client-facing feature, differentiates Rolliworks from competitors, leverages accumulated data)

**Dependencies:**
- D-021 (photo storage index) — reports pull photos from `shared.intake_photos`
- D-022 (watch assembly state model) — reports reference piece-level service history
- New shared test results table (per A-20260629-003) — reports pull RT test data
- SPEC-010 (RolliTime ↔ RS contracts) — determines data availability
- AI report generation infrastructure — model choice, prompt engineering, template design

**Deferred:** Model selection, report format, distribution mechanism (email PDF, portal link, printed), consent/privacy handling. Future SPEC.

### Variance notifier at Receive Watch page (W-41)
**Source:** A-20260629-007 (Q-011-A)
**Description:** When staff at the Receive Watch page (Stage 2 of D-019) find that actual received items don't match the estimate's item count (more or fewer items than expected), the current workflow requires them to leave the page, modify the estimate, then restart the receive process. This is friction-heavy and error-prone.

Should become an inline variance notifier that:
- Detects mismatch between actual received items and estimate expected items
- Displays a clear warning ("Expected 2 items on estimate #4712, but you're receiving 3 pieces")
- Offers a resolution path directly:
  - Edit the estimate inline (add/remove line items, adjust quantities) without leaving the Receive Watch flow
  - OR acknowledge the variance and proceed (with the variance logged to the audit trail per D-020)
- Blocks Stage 2 commit until the variance is resolved

**Priority:** Medium-high (operational friction removed, D-020 audit trail preserved)

**Dependencies:**
- D-019 amended two-stage intake (defines Receive Watch as commit layer)
- D-020 (silent failure banned) — variance events must be logged
- SPEC-001 (two-stage intake schema) — variance notifier UX included in scope

### Cron inventory D-020 audit — maintenance-hourly and vault-storage-fee-daily (W-42)
**Source:** A-20260629-001 (Q-005-C pg_cron discovery)
**Description:** The Q-005 pg_cron discovery revealed six active cron jobs. Two need D-020 silent-failure audit because their scope isn't documented and their failure modes could have real operational impact:

1. **`maintenance-hourly`** — runs every hour on the hour, purpose undocumented in discovery. Could be cleanup, aggregation, cache refresh, or something more consequential. If silently failing, cascading effects possible depending on scope.

2. **`vault-storage-fee-daily`** — bills customers for vault storage. If silently failing, direct revenue impact (uncharged storage fees). High-priority audit.

Both should be inspected for:
- What operation they perform
- Whether they write to a run log with success/failure/started status
- Whether failures produce observable telemetry (banner, email, log entry)
- Whether they follow chunked + idempotent patterns per D-020

Findings should be captured as either new PROD-FIX entries (if bugs are found) or amendments to Q-005 discovery.

**Priority:** Medium (both are running silently now; unknown whether they're broken)

**Dependencies:**
- D-020 (silent failure banned)
- Q-005 discovery (initial cron inventory)

### Fast lane on Shop Floor + concierge queue UI (W-44)
**Source:** Session 2026-07-01 — operational reality of small-jobs workflow
**Description:** The rebuild's Shop Floor dashboard (W-33 visual inventory + W-37 drag-and-drop GUI) must display a fast lane for concierge-managed small jobs alongside the full-service lane. The fast lane has fewer stations and simpler custody, but the same data model discipline, audit trail, and Transferability Test compliance as the full lane.

**Fast lane layout on Shop Floor:**

FAST LANE (Concierge) [Queue: N] → [Concierge: name] → [Tech Bench: name] → [Safe / Client]

Cards move between stations via drag-and-drop or via UI transitions at intake/handoff points. Concierge owns the queue and sequences jobs to techs. Techs perform the work at their bench. Concierge returns to client or routes to safe.

**Concierge queue as first-class UI:**

Separate from Shop Floor viewer but linked. Concierge-role staff work primarily from this queue during their shift. Features:
- Current queue with wait times
- Photo requests from clients (subtype of fast-lane work)
- Conflict management (customer complaints, quality disputes)
- Quick-note field per job
- Tech assignment interface

**Login-based custody transfers:**

Each transition (client → concierge, concierge → tech, tech → concierge, concierge → safe/client) captures `to_staff_code` from currently-authenticated user session. `from_staff_code` selected from dropdown of active staff (sourced from RGTime cache per D-026). Bench-based path never enters safe unless job completes at end-of-day or client is not present.

**Priority:** High (facility-opening critical path — concierge role starts at facility opening if not before)

**Dependencies:**
- D-024 (workflow type first-class + fast lane architecture)
- D-026 (RGTime staff master + login-based custody)
- Canonical data model (jobs table with workflow_type column)
- Shop Floor viewer (W-37) — fast lane rendered as second lane
- Chain-of-custody baseline (D-015) — bench-based path supported
- Cross-app auth (Q-010) — concierge role permissions
- Photo storage index (D-021) — minimum 2 photos per fast-lane job

**In scope for 1/31/2027 facility opening:**
- Fast lane on Shop Floor
- Concierge queue UI (basic queue + assignment + photos + notes)
- Login-based custody transfers for fast lane
- State machine for fast lane per D-024
- Concierge role in auth model
- Concierge staff record in RGTime

**Deferred to Q2-Q4 2027:**
- Photo request queue advanced features (client communication automation)
- Concierge KPI dashboards
- Conflict management workflow (escalation paths, resolution tracking)
- Cross-lane view (jobs that started fast but escalated to full workflow)

**Companion items:** D-024, D-025, D-026, D-015, D-019, D-021, W-33, W-37, W-45

### Tech breadcrumb on sales orders + production reporting (W-45)
**Source:** Session 2026-07-01 — Michael identifies existing workaround (tech name as QBO product code)
**Description:** Currently, tech attribution for production reporting is captured by swapping the QBO product code to the tech's name and pasting the service description back into the line item body. This requires manual work on every sales order, is fragile, and doesn't support multi-tech jobs cleanly.

Replace with:
- **Internal tech attribution** via `job_tech_involvements` join table (job → staff_code, populated automatically from custody transfers once chain-of-custody scanning is live per D-024 and D-015)
- **Sales order breadcrumb field** (`techs_involved` — array of staff_code values) that pulls from the join table when the sales order is created
- **QBO clean push** — product codes go back to service descriptions. Tech attribution stays in RS.
- **Production KPI reports** query the join table directly instead of parsing QBO product code strings

**Staff identity source:** All `staff_code` values reference RGTime (per D-026). Local staff cache in RS provides name/role display without cross-project queries at report time.

**Why breadcrumb over other approaches:**
- QBO doesn't support custom fields on line items reliably; won't push tech attribution cleanly
- Multiple techs per job are supported natively (join table can hold N staff_codes per job)
- Historical accuracy preserved even if a tech leaves or is renamed
- Warranty investigations become a database query, not archaeology on invoice descriptions
- KPI reports work cleanly against structured data

**Zero staff burden:** techs are already recording custody transfers per D-024 and D-015. The breadcrumb aggregates data already being captured.

**Priority:** High (real ongoing time waste + production reporting depends on it)

**Dependencies:**
- D-024 (fast lane + custody recording) — provides automated data source
- D-015 (chain of custody) — same source; full-workflow custody transfers also populate breadcrumb
- D-026 (RGTime staff master) — provides staff_code identity
- Canonical `jobs` table + `job_tech_involvements` join table (part of SPEC-002)
- QBO sync layer (Q-005 findings) — pushes clean product codes only

**In scope for 1/31/2027 facility opening:**
- job_tech_involvements join table
- Automatic population from custody transfers
- Sales order breadcrumb field
- QBO push cleaned up (product codes revert to service descriptions)
- Basic production report (jobs per tech per week)

**Deferred to Q2-Q4 2027:**
- Advanced KPI dashboards
- Warranty rate per tech reporting
- Comparative analytics (this tech vs team average)

**NOT implemented in current Lovable:** Per D-025, this is not an emergency fix. The current workaround (tech name as product code) continues until rebuild ships. Time-waste cost accepted through cutover.

**Companion items:** D-024, D-025, D-026, D-015, D-020, W-44

| W35 | `[BACKLOG]` | **PO split for back-orders.** Accept received items; split outstanding items into a new separate PO. Today PO stays open with received + outstanding mixed. *(Workflow Q&A Tier 2.)* |
| W36 | `[BACKLOG]` | **Backfill flow for items received without estimates.** Today Vianna sets aside, Mike creates estimate after the fact; need a real receive-without-estimate process. *(Workflow Q&A Tier 2.)* |
| W41 | `[BACKLOG]` | **Pickup Station IP cam / Nest integration.** After QR scan at pickup, auto-snap photos of customer leaving with item; audit trail for completed pickups. *(Workflow Q&A Tier 5.)* |
| W45 | `[BACKLOG]` | **Document attachments part of RS → QBO push.** Today added manually to QBO; should be automated part of SO → QBO flow. *(Workflow Q&A Tier 6.)* |
| W46 | `[BACKLOG]` | **Appraisal auto-populate from sales order.** Most appraisal fields already on SO (photo, dial, bracelet, value); auto-populate + ask for extras only when needed. *(Workflow Q&A Tier 6.)* |
| W48 | `[CONSTRAINT]` | **Photo galleria — date-based folders.** Organize client photo gallerias by date; galleria becoming unwieldy at scale. *(Workflow Q&A Tier 6; complements W24 Est# grouping.)* |

---

## RolliWorking (RW)

| ID | Status | Item |
|----|--------|------|
| W26 | `[CONSTRAINT]` | **Inspection photo module on RW (shared with RS).** Same inspection-photo module that exists on RS, added to RW; add movement / back-of-dial / case-back photos to the client record; RW and RS users each need access to photos taken on the other. **Watchmakers need read access** to inspection photos for damage comparison and **write access** for post-disassembly photos. *(Workflow Q&A Tier 3.)* |
| W27 | `[CONSTRAINT]` | **Watchmaker-room photography station.** Special photo station inside the watchmaker room on RW with its own user level; photographs dial removed front & back, plus back of movement; feeds the authenticator app and the client record on RS. **Outside-of-watch photos** needed — current inspection photos taken through crystal. Purpose: protect the watchmaker from blame for pre-existing damage, document back-of-dial engravings, keep movements/dials from being swapped. *(Major byproduct source — feeds AUTH + dial-photo library.)* |
| W29 | `[CONSTRAINT]` | **QR-driven watchmaker bench workflow.** QR codes per process (in-testing, downgrade, parts-request); scan ref-serial first then process QR; fast and gesture-friendly for iPad; replaces drop-down menu. *(Workflow Q&A Tier 2; see D-016.)* |
| W30 | `[CONSTRAINT]` | **Voice-to-text job notes in RW.** Job notes and comments missing from RW today; all staff using RW should contribute; voice-to-text especially needed for iPad. *(Workflow Q&A Tier 2.)* |
| W33 | `[BACKLOG]` | **Bulk "testing complete" email.** Scan all finished jobs from RW; one button → bulk send testing-complete email; then final inspection → SO → invoice + payment link. *(Workflow Q&A Tier 2.)* |
| W37 | `[BACKLOG]` | **Parts request improvements.** Parts name lookup (WM vocabulary ↔ RS vocabulary); auto-suggested price for Mike based on history; on-hand quantity visible in request flow. *(Workflow Q&A Tier 3.)* |
| W42 | `[CONSTRAINT]` | **Shop Floor service-level completion fix.** "In safe awaiting component" already designed but not functional; band finished while watch isn't; advance band room totals; "awaiting invoicing when watch finishes" queue. Fix Shop Floor — not a new feature. *(Workflow Q&A Tier 5.)* |

---

## RolliTime (RT)

| ID | Status | Item |
|----|--------|------|
| W28 | `[CONSTRAINT]` | **Caliber/ref data captured before RT.** Ensure caliber data on the ref & caliber exists before the watch reaches RT for testing; add a toast/prompt reminding to input this before testing. *(Upstream fix lives in RS, but RT must handle the missing-data case gracefully — design for it. Maps to RT's open question on ref# not in CSV.)* |

---

## Cross-app / Inspection & sales-order flow (RS ↔ RC ↔ QBO)

| ID | Status | Item |
|----|--------|------|
| W1 | `[BACKLOG]` | **OCR capture onto client / watch record.** OCR the timing sheet & timing-test results; write captured data directly onto the watch's client record. *(Note: overlaps RT's own Witschi OCR — keep the data-shape consistent.)* |
| W2 | `[BACKLOG]` | **Streamlined verification report (inline, pre-sales-order).** Show a verification report inline before a sales order is created; make verification part of a more streamlined process. |
| W3 | `[BACKLOG]` | **Capture tolerance report & pressure testing.** Capture the tolerance report and timing-sheet pressure-testing results. |
| W4 | `[CONSTRAINT]` | **Control panel to adjust the flow.** A control panel so the workflow/sequence can be altered as needed. *(Implies workflow config should be data-driven, not hardcoded — design for configurability.)* |
| W7 | `[CONSTRAINT]` | **Package scan video traceback.** Back-track from package-scan video footage to track lost items. *(Partial coverage of W49 chain-of-custody vision — extend to full safe audit + station-tagged camera logs.)* |
| W8 | `[BACKLOG]` | **Test document upload on sales order.** Upload test documents on the sales order; send to QBO; store on RS. |
| W9 | `[BACKLOG]` | **Magic link to clients on inspections.** Send a magic link to clients on inspections. |
| W11 | `[CONSTRAINT]` | **Intake photos added to TRC.** Add intake photos to TRC. *(Touches the photo-asset / cross-app model — design so intake photos are a shared, queryable source.)* |
| W12 | `[CONSTRAINT]` | **Send inspection photos from RS to RC.** Select-photos-and-send capability missing today. *(Cross-app photo flow — design the photo library + contracts so RS→RC sharing is a clean read. Workflow Q&A Tier 6.)* |
| W13 | `[BACKLOG]` | **Inspection sending: remove mailto, add preview.** Eliminate the mailto path for inspections; add a preview to see the inspection page before sending. |
| W32 | `[CONSTRAINT]` | **Concierge role + workflow.** New position planned for rush, small, and outsourced jobs that currently fall through the cracks; assignable workflow distinct from main bench flow. *(Workflow Q&A Tier 2.)* |
| W39 | `[BACKLOG]` | **KPI / performance reporting framework.** Tech rating using: time holding job, warranty count, work by dollar amount, demerits for late jobs, RolliTime data signals; end-of-month/quarter cadence. *(Workflow Q&A Tier 4.)* |
| W40 | `[BACKLOG]` | **Time tracking integration.** No current clock-in/clock-out; not required in-suite — integrate separate time-tracking app; tie to KPI framework (W39). *(Workflow Q&A Tier 4.)* |
| W43 | `[CONSTRAINT]` | **Notes field on RC visible across RS + RW.** Single source of truth for special-handling context (e.g. "dead grandfather's watch", "needs done by 9/1/26 wedding"). *(Workflow Q&A Tier 6.)* |
| W49 | `[CONSTRAINT]` | **Chain of custody / theft prevention infrastructure.** Mike's #1 operational risk. IP cameras tied to safe and incoming packages; every label scan logs to queryable security camera system tagged with station ID; backtrack any package scan to video; bulk audit scan of everything in safe storage; reveals orphan/awaiting components; theft deterrence via auditable system. *(Workflow Q&A Tier 1; see D-015.)* |

---

## Authenticator app (AUTH — future module)

| ID | Status | Item |
|----|--------|------|
| W5 | `[CONSTRAINT]` | **Authentication app + inspection-photo integration.** Build pathways to integrate the upcoming auth app into the inspection photos; add more photography stations and features into the workflow. *(Design the photo capture + library now to be a clean AUTH data source — quality + metadata.)* |
| W25 | `[CONSTRAINT]` | **Authenticator app placeholders & integration.** Create placeholders for the auth app; it needs access to intake photos; must return a % authenticity and name the variant type (e.g. dial inserts). *(Reserve the seams now — photo access contract + the placeholder for auth results on the client record.)* |

---

## Notes & open questions on the wishlist itself

- **Byproduct watch:** the highest-value `[CONSTRAINT]` items cluster around **photos** (W5,
  W11, W12, W24, W25, W26, W27, W48) and **conversation/threading** (W10, W16, W17, W44).
  Chain of custody (W49) is the highest-priority operational risk.
- **"TRC" (W11)** — confirm what TRC refers to (typo for RC? a specific module?) before this
  item is actioned.
- **Jarvis / M3KE tie-ins** (W21, W32) — keep these as reserved seams; don't build the modules,
  just don't foreclose them.
- **Offline mode?** (Workflow Q&A open question) — 100% reliance on systems today, no fallback.
  Decide whether to invest in offline workflows or accept the dependency.
- **Concierge + M3KE / Jarvis?** (Workflow Q&A open question) — as rare-job specialist, concierge
  would benefit from AI agent support; worth scoping when W32 is actioned.

---

## From Q-001 answers (2026-06-28)

### Estimate quote improvements (W-29)
**Source:** A-20260628-001 (multi-estimate context)
**Description:** Several improvements to the estimate quote workflow that would save staff time:
- **Separated client notes field** — client notes and concerns should live in a dedicated field, not pasted into the line items. Currently c+p workflow wastes time and pollutes line item data.
- **Separated shipping/drop-off/pick-up field** — these should be their own field so that part number search doesn't return every shipping product when searching unrelated terms.
- **Client asset as separate line item type** — the client's incoming watch (e.g., "1601-38392") should be a separate line item type, not mixed with parts. Currently queries are noisy.
- **Categorization** — line items need to be categorized so search returns relevant results, not everything.
**Priority:** Medium (affects every estimate, time savings compound over hundreds per month)

### Prevent duplicate client assets in inventory (W-30)
**Source:** A-20260628-001
**Description:** Currently, the same client asset (e.g., a specific watch by ref + serial) can be entered into inventory multiple times. This creates duplicate records and confusion downstream. Need validation on intake that prevents duplicate entry of the same physical asset.
**Priority:** High (data integrity)

### Multi-item intake auto-numbering (W-31)
**Source:** A-20260628-002
**Description:** When intake involves multiple watches or multiple components, add an "add another item" button that auto-numbers each item (1 of 2, 2 of 2, etc.). Each numbered item gets its own identifier even when grouped under one estimate, so components from different watches can be tracked independently.
**Priority:** High (correctness of received-component tracking)

### Husband/wife related-account handling (W-32)
**Source:** A-20260628-001
**Description:** When two related clients (typically spouses) ship together on one tracking number with separate accounts and separate estimates, the system needs a way to link the related accounts at intake — not just at billing. Currently this requires manual workarounds.
**Priority:** Medium (occurs regularly, current process is brittle)

### Visual client asset inventory layer (W-33)
**Source:** A-20260628-005
**Description:** Build a visual layer to see all client assets currently in possession: what's in the safe, what's at which station, what's with which watchmaker, what's released. Includes search by client, by ref, by serial, by status. This is the operator-facing surface for the chain-of-custody system (D-015).
**Priority:** High (theft prevention, operational visibility — Michael's #1 risk per workflow capture)

### Intake-to-inventory audit log (W-34)
**Source:** A-20260628-003 + D-019
**Description:** The two-stage intake pattern (Drop-off/Shipping Receive → Receive Watch) requires an audit log that records every disagreement, override, and confirmation between Stage 1 (possession) and Stage 2 (verification + inventory commit). Searchable by date, client, staff, asset. Foundation for theft prevention and accountability.
**Priority:** High (D-015 dependency, audit requirement)

---

_End of wishlist. Living document._
