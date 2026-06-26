# Vianna — Daily Workflow (Consolidated)

**Role:** Front-of-house operations. Primary RolliSuite (RS) intake + RolliConnect (RC) user. Handles all client communication, shipping, intake, work order creation, and watch storage.

**Sources merged:**
- Workflow Q&A (Workflow-QA conflicted copy, 2026-06-25)
- "Vienna — Daily Workflow Map" outline (handwritten, undated)

**Status:** Vianna's role captured. Mike's role captured separately. Watchmaker iPad pain points pending.

---

## 1. Daily Flow

### Morning
- Check **RS Daily Hit List** for the day's triage
- Check **RolliConnect** for new client messages
- **Assign jobs to watchmakers**, set up watchmaker benches
- Open RS Ship Station and shipping label requests
- Email clients with FedEx pickup info for outbound shipments
  - Looks up nearest FedEx ship station based on client's return address
  - Currently requires a separate app to find this — should be done in RS
  - Auto-populate residential vs commercial detection
  - Liability waiver needed for residential ship-to requests
- Process incoming shipping label requests (client fills out form on HTML estimate)
  - Create label manually in IFS Portal (insurance company — no API)
  - Save as PDF, attach via shipping label page
  - OCR detects tracking number, adds to email template
  - **Missing:** no way to know if client used label (needs label audit page + IFS invoice reconciliation)

### Mid-Morning
- Create outbound and inbound shipping labels
- Client replies on RC and email — initial pass of incoming questions
- Initial scan of incoming packages

### Afternoon
- Create work orders (3-color carbon, hand-written)
- Go through troublesome jobs / stagnant jobs
- Prepare outbound shipments
- Answer client update questions via phone, email, RC
- Assign jobs to techs and receive finished jobs back
- Advance jobs forward through departments
- Handle client edits to jobs mid-stream (RS and RW poorly set up for this)

### End of Day
- Set up polish room for Fabian
- Open and process incoming packages (RS Intake → Receive Packages)
  - Scan tracking number → scan estimate → photo intake → email client receipt
- Get jobs ready for polish room
- Secure watches in safe

### Recurring throughout the day
- Customer calls
- Respond to and mark emails for response
- Find and update clients about ongoing work or issues
- Update work orders
- Look up estimates on RS, look up job progress on RW

---

## 2. Tools Used (with frequency)

| Tool | Frequency | Used for |
|---|---|---|
| RS Daily Hitlist | Daily AM | Morning triage |
| RS Intake — Shipping Labels | Daily AM | Inbound labels |
| RS Ship Station | 2x daily | Outbound shipments |
| RS Intake — Receive Packages | Daily PM | Receiving packages |
| IFS Portal | Daily AM | Shipping labels + supply orders (no API) |
| RW Client Replies | 2-5x daily | Updating work orders |
| RW Work Queue | Multiple daily | Job assignment, client updates, communication |
| RS Customer Profiles | Multiple daily | Client communication + troubleshooting |
| RolliConnect | Multiple daily | Client communication + troubleshooting |
| Outlook | Multiple daily | Client communication |

**Top 3 used:** Estimates on RS, Work Queue on RW, Chat on RC

**Most troublesome:** Parts requests on RW

---

## 3. Service Request Flow (Estimate → Intake → Handoff)

### Estimate-first pattern
- **99% of estimates are sent BEFORE the watch arrives** (job templates power most of these)
- Customer fills out request → quote sent → client approves → watch ships in or is dropped off
- Occasionally items arrive without an estimate — estimate is then sent after arrival

### Intake — shipped items
- Scan packages before opening
- Later: open package, scan tracking number, scan estimate number → triggers camera for intake photos
- Estimate isn't a manifest — it's a fuzzy idea of what's coming (clients add/remove items, hand-write notes about changes)

### Intake — in-person drop-off
- Pull up client's estimate
- Accept item, take photo (emails client a drop-off receipt with photos)
- Sometimes print a drop-off receipt
- Item goes into safe awaiting processing
- A front-desk staff member (manager-level) processes the drop-off

### Vianna's processing step
- Reads the estimate
- Completes a physical 3-color carbon work order (hand-written)
- Captures: client name, estimate #, ref-serial (if watch), item details, notes about the job

### Mike's receive-watch step
- Compares what was received vs what RS shows as received (often disagree — Vianna enters correctly but shows wrong on receive watch page)
- Enters W / B / P / PM department tags
- Reads job estimate (shown at bottom of page) to verify nothing missed
- Enters ref-serial, band description
- Prints PDF417 label + watch labels (split-flow jobs get both barcode types)
- Goes to inspection camera page
  - 2 photos via Ipevo camera → switches to microscope camera
  - Triggered by scanning barcode or estimate barcode
  - Photos added to client record
  - **Problem:** photos are messy, need date-based folder organization
  - Some clients have 30-50 watches over years × 10-40 photos each = galleria getting unwieldy

### Approval → in-queue → watchmaker assignment
- Approval = client accepts inspection notes + T&C
- Vianna sets job to "in queue" status, emails client
- Sorted into department bins:
  - Polish only → polish bin
  - Bracelet work → bracelet bin
  - Movement service → watchmaker bin
  - Complete watch (no bracelet work) → entire watch to watchmaker bin
  - Split flow → watch to watchmaker bin, bracelet to band room bin
- **Shop Floor is supposed to handle split flow visibility — it has never worked correctly**

### Watchmaker assignment
- Vianna physically hands watch to watchmaker after RW assignment
- Currently: nothing for watchmaker to do until parts request or status change
- **Future requirement:** watchmakers take dial photos front + back before starting work (feeds Authenticator app, documents pre-work damage)
- Current inspection photos are taken through the (often scratched) crystal — outside-of-watch photos are needed

### Watch storage
- First: bin inside safe, arranged by due date
- Once assigned: each watchmaker has their own bin
- Bins go into safe at night, retrieved each morning by manager
- **Missing:** visual layer showing what every staff member has in their name + how many days they've held it

---

## 4. Pain Points

### From Vianna's outline
- RW not up to date with current job progress or assigned tech
- Transition when a client adds something to their order after initial intake
- Cannot assign incoming packages to multiple estimate numbers

### From Vianna's Q&A
- **Shop Floor** is a mess. Failed module. Separate docs exist explaining how it's supposed to work (won't retype here — see separate Shop Floor doc when needed)
- **Parts requests** are a mess — additional parts requests made as the job develops get lost. Have to look up every parts price manually when the price is already in the database. Should see suggested prices. Need a parts lookup window on the parts request page. Watchmakers use their own names for parts (different from RS names) — need name-aliased search
- **No way to send inspection photos to clients** from RS to RC (need select-photos-and-send capability)
- **No way to add email reply tracking** to RC (need CC/BCC capability so replies land in RC thread)
- **RC adds clients more than once** — conversations get fragmented. Need merge/split + labels
- **QBO document attachments** are added manually. Should be part of the RS → QBO push
- **Appraisal is clunky** — most fields needed are already on the sales order. Should auto-populate (photo, dial, bracelet, value + ask for extras if needed)
- **Human knowledge is wasted** — caliber, bracelet types, dial styles, parts, jobs, appraisal data all get typed but don't accumulate as a knowledge base. M3KE should mine all of it. Next appraisal should suggest dial options from past entries. RolliTime should know a 1601- has a 1570 movement
- **Data silos** — same data typed multiple times. Brand entered repeatedly even though the shop only works on Rolex and Tudor

### From Mike (relevant to Vianna's flow)
- Update emails currently say "system generated" / noreply, but clients reply out of courtesy → creates action items
- Need a "bump email" from a different domain when client emails go to junk
- Pink RC messages (client last reply) and flagged messages = the action queue. Color-coding helps
- Need a notes field on RC visible across RS + RW (e.g., "dead grandfather's watch", "needs done by 9/1/26 wedding")

---

## 5. Workarounds

- Write instructions on work orders (paper, hand-written)
- Small jobs and warranties tracked on sticky notes (no good process)
- Lovable AI or Claude used to remove duplicate jobs or find workarounds
- When Shop Floor blocks adding a job, use Work Queue page to bypass
- Slack used mainly as reminders (used less these days)

---

## 6. Handoffs

- Watches physically handed to staff + RW assignment OR sticky note
- Vianna ↔ staff: in-app + in-person
- **Physical handoffs rarely fail** — the system recognition is what fails
- Shop Floor missing components, Work Queue duplicates, blockers preventing "complete" status

---

## 7. Wishlist

### From Vianna's outline
- Way to ask watchmakers for updates / questions on their timeline
- Area to track troublesome jobs
- Area to track outsourced work
- Easier invoice resend

### From Vianna's Q&A
- More intuitive system — accumulate knowledge from every input
- M3KE as the AI agent that learns Rolex — feeds RS, RW, RC
- Better job logs with date-stamps for every job phase + voice-to-text notes from staff (iPad-driven)
- Blocker messages with due dates
- Better UI for jobs with multiple items
- Better parts ordering logic — recommended vendors + pricing not visibly working

### From Mike (relevant to Vianna)
- Bulk update email send
- Less manual update-email flow
- Inspection photos sendable from RS to RC
- Email CC/BCC into RC threads
- RC client merge/split + labels
- Document attachments part of RS→QBO push
- Auto-populated appraisal from sales order

---

## 8. Edge Cases — STILL OPEN

These were blank in the Q&A and aren't covered by the outline:

- ❓ A recent unusual situation that didn't fit normal flow — what did you do?
- ❓ Customer wants to add or change something after watch is already in process — what happens?
- ❓ App is down or unreachable — fallback?

**Action:** revisit with Vianna when convenient.

---

## 9. Cross-Role Notes

- "RepairShopr doesn't exist" (per Mike) — no naming collision in practice
- Fabian operates the polish room (set up by Vianna at EOD)
- IFS Portal is the shipping insurance vendor (no API — manual workflow)
- Daily Hit List is the central morning triage screen on RS
- Work Queue is the central job assignment screen on RW

---

_End of consolidated workflow document. Update as Vianna's role or workflow changes._
