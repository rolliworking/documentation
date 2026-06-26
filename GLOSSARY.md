# Rolliworks — Glossary

**Purpose:** The canonical vocabulary for the Rolliworks ecosystem. Every term defined once, used identically across code, schema, docs, and conversation. This is the implementation of the naming side of D-010 (canonical ecosystem data model).

**How to use:**
- Before introducing a new term anywhere (code, doc, conversation), check this glossary
- If the term exists, use the exact spelling and meaning here
- If the term is new, add it here first, then use it
- If you disagree with a definition, update the glossary first, then update everything else

**One-line goal anchor:** _[paste the rebuild one-line goal here]_

**Source:** Workflow Q&A updates applied 2026-06-25. See `workflows/MICHAEL-WORKFLOW.md` and `workflows/VIANNA-WORKFLOW.md`.

---

## Apps & Systems

**RolliSuite (RS)** — the ERP application. Source of truth for client files, customer data, caliber and reference data. Lives at rollisuite.com. "RS" in conversation and docs means RolliSuite.

**RolliWorking (RW)** — the workshop/workflow application. Watchmaker bench tool. Tracks job status, watchmaker assignment, parts approval. Lives at rolliworking.lovable.app.

**RolliConnect (RC)** — the CRM and customer portal application. Customer-facing communications. Lives at my.rolliworks.com.

**RolliTime (RT)** — watch accuracy and power-reserve testing application. Owns testing data + the dial-photo library. Separate database. Integrates via API contracts.

**Authenticator (AA)** — genuine-vs-counterfeit verification app. In ecosystem; reads from dial-photo library and intake photos.

**M3KE** — internal AI knowledge service. v1 live as escalation tool. Future: chatbot embedded in RolliSuite.

**Jarvis** — planned AI module. Scope TBD.

**PartsWiki** — planned standalone parts-pricing app. See `REBUILD-PREREQUISITES.md` and HANDOFF docs.

**RolliCurator** — planned knowledge-harvest wizard module for master table completion. See `ROLLICURATOR-CONCEPT.md`.

**Rolliworks** — the operating watch service center company.

**Rollie Group** — the strategic holding entity / platform brand.

**IFS Portal** — external shipping insurance vendor. No API. Used daily AM for shipping labels and supply orders. Manual workflow today — copy/paste data, save PDFs, attach to RS via shipping label page. Possible future integration target.

---

## RolliSuite Modules & Screens

**Daily Hit List** — central morning triage screen on RolliSuite. First thing Vianna checks each morning. Surfaces highest-priority items.

**Work Queue** — central job assignment screen on RolliWorking (not RS — listed here because paired with Daily Hit List in daily ops). Used multiple times per day for assigning work, client updates, and communication.

**Shop Floor** — RolliSuite module designed for service-level workflow visibility (e.g. "in safe awaiting component"). **Currently broken** — has never worked correctly. Root cause: components not properly registered at intake. Fixing Shop Floor unblocks service-level completion, component-level safe visibility, and split-flow job tracking.

**Pickup Station** — RolliSuite module for completing customer pickups. Scans pickup QR, compares intake photos to item, takes exit photos. Has QBO payment latency bypass for known-paid pickups.

**Receive Watch / Receive Packages** — RolliSuite intake module. Currently fragments caliber, department, ref-serial, and component data across 5 tables (see intake audit). Major rebuild priority.

---

## Watch Domain Terms

**Reference** (or *reference number*, *ref*) — the manufacturer's catalog number for a watch model, e.g. `1601`, `16710`. Identifies the *model*, not the individual watch. Format examples: `1601`, `16710BLNR`, `116610LN`.

**Serial** (or *serial number*) — the manufacturer's unique number for an individual watch. Identifies *this specific watch*, not its model.

**Ref-serial** — the combined identifier in the form `[reference]-[serial]`, e.g. `1601-1442547`. The universal join key across all systems (per RolliTime contract §7).

**Caliber** — the movement model, e.g. `1570`, `3135`, `3235`. Multiple references can share a caliber; some references support multiple calibers.

**Caliber family** — a grouping of related calibers, e.g. `31xx` includes 3130, 3135, 3155, etc. Used when the exact caliber isn't known at intake. *Should be promoted to specific caliber via RolliCurator workflow.*

**Dial** — the face of the watch. Many variants per reference (color, lume type, era).

**Bracelet** — the metal band, e.g. Jubilee, Oyster, President. Multiple variants per reference (era, end-link size).

**Movement** — alternate name for the caliber's internal mechanism. *Prefer "caliber."*

**Request** — a customer's incoming inquiry, before an estimate is created.

**Estimate** — a priced quote sent to the customer. Defines what work will be done. 99% of estimates go out before the watch arrives.

**Job** — the actual work being performed on a watch.

**Work Order** — physical 3-color carbon paper, hand-written by Vianna, kept with components per department. **NOT a software concept.** Do not model in the rebuild.

**3-Color Carbon Work Order** — same as Work Order. Each department takes a copy and keeps it with components; different colors per department. The *information captured on it* must be in the digital workflow, but the paper itself is not modeled.

**Customer / Client** — interchangeable. Both terms used by all staff. No software distinction.

**Sales Order (SO)** — created after final inspection. Triggers invoice + payment link.

**Shop work order (SWO)** — internal vendor/outbound work order in RolliSuite (software concept). Distinct from the physical 3-color carbon Work Order. Has its own QBO flow (see WISHLIST W18).

**Verification report** — the post-service document confirming testing results. Currently OCR-generated; future versions consume structured RolliTime data.

---

## Data Architecture Terms

**Canonical data model** — the single agreed-upon vocabulary + table structure used across the entire ecosystem. See D-010.

**Master table** — a table that holds the authoritative version of a shared entity (customers, references, calibers, bracelets, dials, parts). Living. Multiple apps read; designated owner writes/promotes.

**Living master** — a master table that grows and improves over time through contributions from multiple apps and humans. See D-011.

**Stub row** — a master row that exists but isn't complete. Has identifying info (e.g. caliber number) but lacks specs, tolerances, validations. Promotion to `complete` requires human review.

**Promotion** — the workflow that moves a row from `stub` → `partial` → `complete` → `validated` → `authoritative`. See D-011 and RolliCurator concept.

**Source of truth** — the system designated as authoritative for a given entity. E.g. RolliSuite is source of truth for caliber/reference data (D-002).

**Reserved schema slot** — an empty table or column structure created during the rebuild to anticipate a future integration. See `FUTURE-INTEGRATIONS.md`.

**Shared schema** — Postgres schema name `shared.*` containing entities all apps need (customers, watches, references, calibers, bracelets, parts).

**App schema** — Postgres schema names `rs.*`, `rw.*`, `rc.*` containing entities specific to one app.

**AI module schema** — Postgres schema `ai_modules.*` containing outputs from AI modules (embeddings, classifications, interactions, escalations).

---

## Department Codes

(Used at intake and throughout)

**W** — Watchmaker department.
**B** — Bracelet department.
**P** — Polish department.
**PM** — Project Management.

Stored today as four booleans on `client_property` (`dept_w`, `dept_b`, `dept_p`, `dept_pm`). Currently not exposed to RolliWorking — see intake data audit.

---

## Status Terminology

Confirmed in use today on RolliWorking (more to be added via system mapping):

| Status | Meaning |
|--------|---------|
| **In Progress** | Actively being worked on |
| **In Testing** | At RolliTime / timing station |
| **Waiting Parts Approval** | Parts request submitted, awaiting client approval |
| **Downgrade** | Moved backward (e.g. testing → in-progress) or warranty rework |
| **Ready for Inspection** | Mike's final inspection step *(implied)* |
| **Ready to Ship** | Post-invoice, awaiting shipment *(implied)* |

Many statuses currently mean different things across RolliSuite vs RolliWorking — reconcile during rebuild. In code/schema use lowercase snake_case (`in_testing`, not `In Testing`).

---

## Roles (People)

**Michael** — owner of Rolliworks. Product owner across all apps. Receives Watch module, inspection, parts pricing approval, final inspection before invoicing.

**Vianna** — front-of-house. Primary RolliConnect + RolliSuite (intake) user. Shipping, intake, work order creation, watch storage.

**Watchmaker Room Supervisor** — supervises bench watchmakers. Primary RolliWorking user. Owns caliber and reference master table promotions.

**Fabian** — polish room operator. Vianna sets up his bench at end of day. Polish-only jobs route to him via a separate bin.

**Concierge** — planned position (not yet filled). Responsible for rush jobs, small jobs, and outsourced work — categories that currently fall through the cracks. Will have a workflow distinct from the main bench flow. Likely benefits from AI agent assistance (M3KE / Jarvis).

**Michael R. Michaels** — incoming partner, CW21-certified, previously Hublot / WatchBox / Govberg / Richemont.

---

## Process Terms

**Intake** — the process of receiving a watch for service. Currently happens via Receive Watch / Receive Packages in RolliSuite + front desk kiosk.

**Drop-off** — sub-flow of intake where a customer hand-delivers (vs. ships).

**Pickup** — customer collecting their serviced watch. Includes signature comparison via Pickup Station; intake photo comparison and exit photos.

**Custody** — chain-of-possession of a watch between receipt and return. Tracked via `client_property` in RS. See W49 / D-015 for planned audit infrastructure.

---

## Conventions

- **RS** = RolliSuite. **RW** = RolliWorking. **RC** = RolliConnect. **RT** = RolliTime. **AA** = Authenticator.
- "Caliber" not "movement" in code/schema (movement is a synonym for human-readable contexts only).
- Reference numbers as strings, never integers (leading zeros matter: `0601` ≠ `601`).
- Times in UTC in database, displayed in local time in UI.
- Status enums are lowercase snake_case in code (`in_testing`, not `In Testing`).
- **Work Order** (physical paper) ≠ **Shop work order (SWO)** (software vendor/outbound flow). Never conflate.

---

## How to add a new term

When introducing a new term to the ecosystem:

1. Add the term to this glossary first with a one-line definition
2. Use it consistently everywhere afterward
3. If the term overlaps an existing one, deprecate the older term and note the replacement
4. Update any code/schema that uses the old term in the next rebuild slice

---

_End of glossary. Living document. Append and refine as the ecosystem grows._
