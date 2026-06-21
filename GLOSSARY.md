# Rolliworks — Glossary

**Purpose:** The canonical vocabulary for the Rolliworks ecosystem. Every term defined once, used identically across code, schema, docs, and conversation. This is the implementation of the naming side of D-010 (canonical ecosystem data model).

**How to use:**
- Before introducing a new term anywhere (code, doc, conversation), check this glossary
- If the term exists, use the exact spelling and meaning here
- If the term is new, add it here first, then use it
- If you disagree with a definition, update the glossary first, then update everything else

**One-line goal anchor:** _[paste the rebuild one-line goal here]_

---

## Apps & Systems

**RolliSuite** — the ERP application. Source of truth for client files, customer data, caliber and reference data. Lives at rollisuite.com. *Never refer to this as "RS" alone in conversation or docs — always say "RolliSuite" or "RolliSuite (RS)."*

**RolliWorking** — the workshop/workflow application. Watchmaker bench tool. Tracks job status, watchmaker assignment, parts approval. Lives at rolliworking.lovable.app.

**RolliConnect** — the CRM and customer portal application. Customer-facing communications. Lives at my.rolliworks.com.

**RolliTime** — watch accuracy and power-reserve testing application. Owns testing data + the dial-photo library. Separate database. Integrates via API contracts.

**RepairShopr** — third-party SaaS used for pricing, billing, and inventory. We consume its API; we do not own its data. *Never refer to this as "RS" alone.*

**M3KE** — internal AI knowledge service. v1 live as escalation tool. Future: chatbot embedded in RolliSuite.

**Jarvis** — early-stage AI module. Scope TBD.

**PartsWiki** — planned standalone parts-pricing app. See `REBUILD-PREREQUISITES.md` and HANDOFF docs.

**RolliCurator** — planned knowledge-harvest wizard module for master table completion. See `concepts/ROLLICURATOR-CONCEPT.md`.

**Rolliworks** — the operating watch service center company.

**Rollie Group** — the strategic holding entity / platform brand.

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

**Service request (SR)** — a unit of work the shop performs for a customer. The primary transactional entity.

**Estimate** — a price quote produced before work begins. Becomes a sales order if accepted.

**Sales order (SO)** — an accepted estimate; work is committed.

**Shop work order (SWO)** — an internal work order, distinct from a customer-facing sales order. Has its own QBO flow (see open items).

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

(To be reconciled across apps — many of these currently mean different things in RolliSuite vs. RolliWorking. Capture as found, normalize during rebuild.)

_(Section to be filled in as workflow mapping uncovers actual status terms used.)_

---

## Roles (People)

**Michael** — owner of Rolliworks. Product owner across all apps.

**Vianna** — front-of-house. Primary RolliConnect + RolliSuite (intake) user.

**Watchmaker Room Supervisor** — supervises bench watchmakers. Primary RolliWorking user. Owns caliber and reference master table promotions.

**Michael R. Michaels** — incoming partner, CW21-certified, previously Hublot / WatchBox / Govberg / Richemont.

---

## Process Terms

**Intake** — the process of receiving a watch for service. Currently happens via the Receive Watch module in RolliSuite + front desk kiosk.

**Drop-off** — sub-flow of intake where a customer hand-delivers (vs. ships).

**Pickup** — customer collecting their serviced watch. Includes signature comparison via kiosk.

**Custody** — chain-of-possession of a watch between receipt and return. Tracked via `client_property` in RS.

---

## Conventions

- Always write **RolliSuite** or **RolliSuite (RS)** — never "RS" alone in shared documents.
- Always write **RepairShopr** in full — never abbreviate.
- "Caliber" not "movement" in code/schema (movement is a synonym for human-readable contexts only).
- Reference numbers as strings, never integers (leading zeros matter: `0601` ≠ `601`).
- Times in UTC in database, displayed in local time in UI.
- Status enums are lowercase snake_case in code (`in_testing`, not `In Testing`).

---

## How to add a new term

When introducing a new term to the ecosystem:

1. Add the term to this glossary first with a one-line definition
2. Use it consistently everywhere afterward
3. If the term overlaps an existing one, deprecate the older term and note the replacement
4. Update any code/schema that uses the old term in the next rebuild slice

---

_End of glossary. Living document. Append and refine as the ecosystem grows._
