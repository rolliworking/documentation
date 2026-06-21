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

---

## How to use this with the pipeline

- When pointing Qwen at an app, have it read this file's relevant section so it documents the
  app already knowing which future items must be accounted for.
- Every `[CONSTRAINT]` item is a note to carry into the rebuild brief for the app it touches —
  "leave a seam for X."
- When triaging what to build next, work from `NORTH-STAR.md` §5 first; this list is the
  parking lot, not the priority queue.

---

## RolliConnect (RC)

| ID | Status | Item |
|----|--------|------|
| W14 | `[IN-SCOPE]` | **Process flow charts for returning clients.** Turn off old process flow charts when clients return; support showing multiple flows for multiple ongoing/old jobs. *(Tied to the current client-portal process-flow priority — handling non-linear states like in-testing/uncased and multiple flows.)* |
| W16 | `[IN-SCOPE]` | **Message routing & request merging.** Improve message routing; add ability to merge requests; resolve one-client-multiple-threads fragmentation. *(Current priority — the message-thread fracturing fix.)* |
| W10 | `[CONSTRAINT]` | **Estimate reply captured to RC.** When a client replies to an estimate, add the conversation to RC. *(Touches the threading model — design the conversation data so estimate replies attach to the right unified thread.)* |
| W15 | `[BACKLOG]` | **Date requests on RC.** Have the request dated. |
| W17 | `[CONSTRAINT]` | **Magic link to RC + client portal connection.** Add a magic link to RC on the inspection-notes HTML page; connect questions/comments to the client portal on their page. *(Touches portal + threading — design seams so portal questions land in the unified conversation.)* |

---

## RolliSuite (RS)

| ID | Status | Item |
|----|--------|------|
| W6 | `[BACKLOG]` | **Improve shop work orders.** Mark work orders complete; export to QBO. |
| W18 | `[BACKLOG]` | **Shop work orders: QBO flow, payment matching, outbound notes.** Invoice-like flow when pushing to QBO; match work-order payments to the bill; leave a note on the estimate page (custom notes) when an item is sent out; scan a label (Est# or customer watch label as Part#) and leave a comment. |
| W19 | `[BACKLOG]` | **Auto-populate client watch ref/serial onto estimate.** Add ref-serial to the client estimate automatically after the receive page is done; eliminate manual entry per job. |
| W20 | `[BACKLOG]` | **Email vendor a shop work order before sending item out.** Use an email template + mailto: function. |
| W21 | `[CONSTRAINT]` | **Daily hit list / to-do lists on RS.** Radio buttons for a curated per-person view + an "All" button for admin; to-do items; add items to someone else's list (e.g. `#vienna order paper`); future tie-in to Jarvis. *(The Jarvis tie-in is the constraint — design the to-do/hit-list data so a future module can read/write it.)* |
| W22 | `[BACKLOG]` | **Better warranty flow** for the watchmaker room and small jobs. |
| W23 | `[BACKLOG]` | **Fix estimate Part# column.** Estimates currently print the description in the Part# column — fix this. |
| W24 | `[CONSTRAINT]` | **Organize trade-account photos by Est#.** Trade accounts accumulate hundreds of photos; organize by estimate number. *(Touches the photo-asset model — design photo storage keyed so this grouping is a query, not a rework.)* |

---

## RolliWorking (RW)

| ID | Status | Item |
|----|--------|------|
| W26 | `[CONSTRAINT]` | **Inspection photo module on RW (shared with RS).** Same inspection-photo module that exists on RS, added to RW; add movement / back-of-dial / case-back photos to the client record; RW and RS users each need access to photos taken on the other. *(Touches the cross-app photo-sharing model — design the shared photo library + access so both apps read/write cleanly.)* |
| W27 | `[CONSTRAINT]` | **Watchmaker-room photography station.** Special photo station inside the watchmaker room on RW with its own user level; photographs dial removed front & back, plus back of movement; feeds the authenticator app and the client record on RS. Purpose: protect the watchmaker from blame for pre-existing damage, document back-of-dial engravings, keep movements/dials from being swapped. *(Major byproduct source — feeds AUTH + dial-photo library. Design the photo capture + metadata now so it's a clean source later.)* |

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
| W7 | `[BACKLOG]` | **Package scan video traceback.** Back-track from package-scan video footage to track lost items. |
| W8 | `[BACKLOG]` | **Test document upload on sales order.** Upload test documents on the sales order; send to QBO; store on RS. |
| W9 | `[BACKLOG]` | **Magic link to clients on inspections.** Send a magic link to clients on inspections. |
| W11 | `[CONSTRAINT]` | **Intake photos added to TRC.** Add intake photos to TRC. *(Touches the photo-asset / cross-app model — design so intake photos are a shared, queryable source.)* |
| W12 | `[CONSTRAINT]` | **Send inspection photos from RS to RC.** *(Cross-app photo flow — design the photo library + contracts so RS→RC sharing is a clean read.)* |
| W13 | `[BACKLOG]` | **Inspection sending: remove mailto, add preview.** Eliminate the mailto path for inspections; add a preview to see the inspection page before sending. |

---

## Authenticator app (AUTH — future module)

| ID | Status | Item |
|----|--------|------|
| W5 | `[CONSTRAINT]` | **Authentication app + inspection-photo integration.** Build pathways to integrate the upcoming auth app into the inspection photos; add more photography stations and features into the workflow. *(Design the photo capture + library now to be a clean AUTH data source — quality + metadata.)* |
| W25 | `[CONSTRAINT]` | **Authenticator app placeholders & integration.** Create placeholders for the auth app; it needs access to intake photos; must return a % authenticity and name the variant type (e.g. dial inserts). *(Reserve the seams now — photo access contract + the placeholder for auth results on the client record.)* |

---

## Notes & open questions on the wishlist itself

- **Byproduct watch:** the highest-value `[CONSTRAINT]` items cluster around **photos** (W5,
  W11, W12, W24, W25, W26, W27) and **conversation/threading** (W10, W16, W17). These are the
  two byproduct streams the suite compounds — worth designing carefully now.
- **"TRC" (W11)** — confirm what TRC refers to (typo for RC? a specific module?) before this
  item is actioned.
- **Jarvis / M3KE tie-ins** (W21) — keep these as reserved seams; don't build the modules,
  just don't foreclose them.
