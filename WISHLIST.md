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
| 3 | Watchmaker bench improvements | W26, W27, W37 |
| 4 | Performance + accountability | W39, W40 |
| 5 | Pickup + handoff polish | W41, W42 |
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

### AI-assisted intake verification — scaffolding now, AI later (W-35)
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
- RolliTime dial photo library — first major consumer of photo discipline (other than W-35 itself)
- Future Authenticator app — likely supplier of the visual-reasoning model

**Status:** Wishlist (scaffolded with D-019 build; AI activation pending tech maturity)

**Related:**
- W-33 (visual asset inventory layer) — UI surface where AI mismatches would surface
- W-34 (intake-to-inventory audit log) — where AI/human disagreements get logged

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
