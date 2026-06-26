# Michael — Daily Workflow (Consolidated)

**Role:** Owner. Operations oversight, financial review, strategic direction, AI/dev coordination across the four-app suite. Receives Watch module, inspection, parts pricing approval, final inspection before invoicing.

**Sources merged:**
- Workflow Q&A original (Workflow-QA, 2026-06-25) — Michael section
- Remaining Workflow Questions (2026-06-25) — Mike + watchmaker observations + cross-role

---

## 1. Edge Cases

### Customer ships own + friend's job on one label
- One shipping label, two estimates inside the package
- **Today:** workaround — Vianna sets one aside, Mike sorts after
- **Need:** ability to assign incoming packages to multiple estimate numbers at intake

### Customer adds/changes after watch is already in process
- **Today:** hand-write changes on the 3-color carbon work order
- **Need:**
  - Add a note to job in Work Queue
  - Note lands on Daily Hit List
  - Assign to a person (watchmaker / Vianna / Mike / concierge)
  - Check off or carry over to next day

### App down or unreachable
- **Today:** no fallback. 100% reliance on the systems
- **Open question:** do we need an offline mode?

---

## 2. Watchmaker Bench (iPad usage)

### Status changes watchmakers make
- In testing
- Waiting parts approval
- Downgrade from in-testing back to in-progress

### How they do it today
- Drop-down menu on RW
- **Want:** QR codes per process. Scan ref-serial (e.g. `1601-743849`), then scan process QR — fast and easy
- **Want:** voice-to-text for job notes and comments (currently missing from RW — all staff using RW should contribute)

### What's bad about the iPad UI
- Basically a web view, not streamlined for iPad gestures
- "UI for iOS is very different than one for desktop"

### Parts request flow (watchmaker side)
- WM types client name or estimate number
- Types parts description
- Submits
- Mike enters price, sends approval request to client
- **Want:** parts name lookup (cross-reference WM naming vs RS naming)
- **Want:** auto-suggested price for Mike based on history
- **Want:** on-hand quantity visible in the request flow

### Other watchmaker complaints
- Need access to inspection photos to compare and look for damage
- Need ability to add photos (dial + hands look different after crystal is removed)
- Current inspection photos are taken through the crystal — outside-of-watch photos needed

---

## 3. Cross-Role Handoffs

### Customer arrives
- In person → Vianna or Mike
- Shipped in → Vianna

### Vianna captures intake → how does watchmaker know there's work?
- They don't, and they don't need to
- Managers manage the Work Queue

### Watchmaker finishes work → how does Vianna know to contact customer?
- Mike sends a "testing complete" email from RW
- **Want:** bulk email page. Scan all finished jobs, one button → bulk "testing complete" email to clients
- After: final inspection → create SO → send invoice + payment link → ship after payment

### Customer pickup
- Manager uses Pickup Station
- Enter SO or estimate # → checks for payment
- If no payment: take payment now
- QBO has payment registration latency → use bypass if payment is known to be made
- Scan pickup QR → compare intake photos to item → take exit photos
- **Want:** IP cam / Nest integration to snap photos automatically after QR scan (audit trail of customer leaving with item)

### Where watches get "lost in the system"
- Item is in the system, but searching the estimate number doesn't pull it up
- Root cause: components aren't registered, so Shop Floor doesn't function
- This is a Shop Floor failure, not a missing concept

---

## 4. Naming and Terminology

| Term | Definition |
|---|---|
| Request | Before an estimate |
| Estimate | Price quote sent to client |
| Job | The work being done |
| Work Order | Physical 3-color carbon paper, hand-written, kept with components by department. **Not relevant to software** |
| Customer / Client | Interchangeable |

- "RolliSuite" is used consistently → no RS naming collision in practice
- "RepairShopr doesn't exist" (per Mike)

---

## 5. Rare but Important Situations

### Rush jobs / small jobs / outsourced jobs
- **Planned:** add a Concierge position to handle these
- These jobs currently fall through the cracks
- Need: assignable to concierge with their own workflow

### Customer disputes or returns
- Doesn't happen much → no formal process

### Watchmaker rejects / redo
- "Downgrade" or "warranty" — existing flow

### Vendor / parts ordering exceptions (back-ordered, wrong part)
- **Today:** accept received items, leave PO open with remaining items
- **Want:** option to split outstanding items into a new separate PO

### System outage workaround
- None. 100% reliance
- **Open question:** offline mode?

### End of month / quarter
- **Want:** technician performance reports
- **Want:** service-level completion (band done while watch isn't) — already designed into Shop Floor as "in safe awaiting component," just not functional today. Fix = make Shop Floor work, not a new feature
- **Want:** "awaiting invoicing when watch finishes" queue visibility

---

## 6. Final Reflection

### Biggest operational risk (per Mike)
**Chain of custody / theft prevention.**

> Not knowing which job which tech has and for how long. Not knowing what's waiting in the safe. No trail or log for chain of custody within the shop.

**Wanted infrastructure:**
- IP cameras tied to safe and incoming packages
- Backtrack package scan → query video footage
- Every scan of a customer label logs to security camera system → queryable by station
- Bulk method of scanning everything in safe storage → audit capability
- Some watches are orphan components, some are awaiting other components — staff visibility prevents theft

### Biggest opportunity
**Better performance reports + KPI framework.**

- Rating system using KPIs:
  - Time a tech holds a job
  - Number of warranties
  - Work done by dollar amount
  - Demerits for jobs done after due date
  - RolliTime data quality signals
- No current clock-in/clock-out system — possibly should add one
- Doesn't have to be built in — integrate a separate time-tracking app if needed

### Anything else
**7-day stagnation tracker.**

- Jobs received but not progressed within 7 days (prior to being in Work Queue)
- Need overview of these
- Note field + reset-7-day-timer capability
- "Trust that staff will hand off for inspection, but need a way to keep them honest"

---

## 7. Vianna — Edge Cases (now answered)

### Items received that don't match estimate
- Vianna sets aside → Mike creates estimate after the fact
- **Need:** method to backfill estimate data when items arrive without estimates

### Customer adds/changes mid-process
- Hand-write on the 3-color carbon work order (same as Mike)

### App down
- None. Hand-written notes only.

---

## 8. Cross-document references

- See `VIANNA-WORKFLOW.md` for front-of-house operations detail
- See `WISHLIST.md` for full feature wishlist (this document adds chain-of-custody, QR watchmaker, KPI framework, 7-day tracker, multi-estimate intake, PO split, concierge workflow)
- See `DECISIONS-REGISTRY.md` for D-015 (chain of custody) and D-016 (QR watchmaker flow)
- See `GLOSSARY.md` for confirmed naming definitions

---

_End of Michael consolidated workflow document._
