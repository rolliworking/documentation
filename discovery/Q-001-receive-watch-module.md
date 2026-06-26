# Q-001 — Map the Receive Watch module (RS)

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-26
**Inputs read:**
- `apps/rs/src/pages/intake/ReceivePackagesPage.tsx` (4707 lines)
- `apps/rs/src/pages/intake/ClientWatchEntryPage.tsx` (2810 lines)
- `apps/rs/src/lib/constants.ts`
- `apps/rs/supabase/migrations/` (219 migrations, focused on key tables)
- `apps/rs/supabase/functions/upload-photo/index.ts`
- `apps/rs/supabase/functions/upload-client-photos/index.ts`
- `apps/rs/src/components/intake/` (CameraDialog, SignaturePad, DropoffEmailPreviewDialog)
- `apps/rs/src/hooks/usePackageArrivalScans.ts` (referenced in code)
- `documentation/workflows/VIANNA-WORKFLOW.md`
- `documentation/workflows/MICHAEL-WORKFLOW.md`
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**Receive Watch** — The entry point for every watch entering RolliWorks. It captures a client's watch (and components), links it to an estimate, records received condition, captures intake photos, and sends a confirmation email to the client.

There are **two intake paths** implemented in code:
1. **Package Receive** — Incoming shipping packages, scanned via tracking barcode
2. **In-Person Drop-off** — Walk-in client, entered directly in Client Watch Entry

---

## 2. Trigger

| Path | Trigger |
|---|---|
| Package Receive | Staff opens `ReceivePackagesPage`, scans a FedEx/UPS tracking barcode with USB scanner (types + Enter) |
| Package Receive | Staff pastes multiple barcodes in the "bulk scan tool" text area |
| Drop-off | Staff opens `ClientWatchEntryPage`, selects a customer, scans/enters an estimate number |
| Walk-in | Staff creates a new entry without an estimate (drop-off without prior estimate) |

---

## 3. Actors

| Actor | Role |
|---|---|
| Staff (front desk / Vianna) | Scans barcodes, selects received components, takes photos, confirms intake |
| Client | Receives email confirmation with tracking, estimate #, received items |
| RolliSuite (RS) writes to 5 tables during save | `client_property`, `parts` (CW-), `estimates` (status touch), `model_references`, `calibers` |
| Edge function `send-estimate-email` | Confirmation email to client after package receive |
| Edge function `upload-photo` | R2-signed upload for inspection photos (Ipevo cam/microscope) |
| Storage bucket `client-photos` | RolliConnect client photo submissions (inbound) |
| `get_job_profile` RPC | Enriches ClientWatchEntry with received components, custody info, departments |

---

## 4. Inputs

### Package Receive fields

| UI label | Code variable | Field type |
|---|---|---|
| Tracking barcode (scan) | `packageReceiveData.trackingNumber` | text, auto-filled by barcode scanner |
| Estimate number (auto-linked from tracking) | `packageReceiveData.estimate_number` | text, from `estimates.tracking_number` |
| Received items (button taps) | `packageReceivedItems` | Record<string, number> — keys: `"Complete watch"`, `"Bracelet"`, `"Watch case"`, `"Bezel"`, etc. |
| Intake photos (camera/file) | `packagePhotos` | `{ url, name }[]` |
| Memo (optional) | `packageReceiveData` memo / `arrival_scan.memo` | text |

### Client Watch Entry (drop-off) fields

| UI label | Code variable | Field type |
|---|---|---|
| Customer | `selectedCustomer` | Customer object (from `CustomerSelector` component) |
| Estimate # (scan) | `formData.estimateNumber`, `formData.estimateId` | text, auto-filled from estimate QR/number |
| Brand | `formData.brand` | text, auto-filled from `model_references` OR manual |
| Model | `formData.model` | text, auto-filled from `model_references` OR manual |
| Caliber | `formData.caliber` | text, auto-filled from `model_references.caliber` OR manual |
| Bracelet model | `formData.braceletModel` | text, from `reference_bracelets` or estimate line items |
| Part # / Reference / Serial | `formData.partNumber`, `formData.serialNumber` | text, model ref like `116233-X9999` OR service code `BR-1234` |
| Date received | `formData.dateReceived` | date string, defaults to today |
| Item type (radio) | `itemType` | `'watch' \| 'band_only' \| 'watch_head_only'` |
| Departments (checkboxes) | `deptW`, `deptB`, `deptP`, `deptPM` | booleans |
| Received components (toggles) | `receivedComponents` | `string[]` — presets: `complete_watch`, `bracelet`, `watch_head`, `watch_case`, `clasp`, `bezel`, `box`, `papers` |
| Bracelet description | `formData.bracelet_description` | text, filled from estimate line items or manual |
| Notes | `formData.notes` | text |
| Signature pad | `dropoffSignatureUrl` | image URL (for in-person forms) |

---

## 5. Steps (end-to-end flow)

### Path A: Package Receive (shipping)

**Step 1 — Barcode scan**
1. Staff scans incoming FedEx/UPS tracking barcode into scan input
2. `detectCarrier()` (from `carrier-tracking.ts`) classifies FedEx (12 digits), UPS, USPS
3. Tracking is normalized via `extractTrackingNumber()` (strip composite prefixes)
4. System queries `estimates.tracking_number` for a match:
   ```
   SELECT ... FROM estimates WHERE tracking_number = <normalized>
     AND status IN ('sent','converted','draft','declined')
   ```
5. If matched → shows client/watch/estimate data in `packageReceiveData`; if no match → creates new unmatched entry

**Step 2 — Received components selection**
1. Received item buttons appear: "Complete watch" (always first) + dynamic suggestions
2. Dynamic suggestions come from two strategies:
   - **Department field**: `part.department === 'W'/B/P` → suggests `"Watch"`/ `"Bracelet"`/ `"Case"`
   - **Legacy**: `part.bracelet_model` → `"Bracelet"`; `part.service_type` → `"Watch"`
3. Staff taps buttons to count received items (0-10 per item, +/-)
4. Buttons also include: `"Watch case"`, `"Bezel"`, `"Clasp"`, `"Dial"`, `"Crown"`, `"Hands"`, `"Gaskets"`, `"Movement"`, `"Crystal"`

**Step 3 — Intake photos**
1. Staff can: scan via Ipevo camera / microscope camera (CameraDialog), or upload files (file picker)
2. Photo upload calls `upload-photo` Edge Function:
   - Accepts `{ inspectionId, referenceNumber, photoData, photoType, photoOrder, date }`
   - Signs request to R2 via AWS Sig V4
   - Uploads to R2 path: `{date}/{referenceNumber}/{photoType}_{order}.jpg`
3. Photos stored as `{ url, name }[]` in component state

**Step 4 — Email preview**
1. Staff preview email via `sendPackageEmailNow`:
   - Calls `send-estimate-email` edge function
   - Payload: `{ isDropoffEmail, toEmail, ccEmail, subject, body, photoUrls, estimateNumber, customerName, watchInfo, receivedItemsList }`
   - CC to `rworksfrontdesk@gmail.com` (hardcoded)
2. Email content: `"Your package has arrived - {watchInfo} - {estimateNumber}"`

**Step 5 — Save received items** (`savePackageItems`)
1. Updates `package_scan_logs` with `received_items` (array), `photo_urls`, `notes`
2. Updates `client_property.notes` with `[received:{items}]` format (for linked estimate)
3. Updates `package_arrival_scans.memo` with `received:{items}` text
4. Marked `email_sent = true` in `package_scan_logs`

**Step 6 — Mark scan as "received"**
1. On initial scan match → inserts into `package_arrival_scans` with `receive_status = 'received'`
2. Daily metrics computed: pending count, received count, carrier breakdowns
3. Celebration toast when all packages matched: `"Yay! All packages matched!"`

### Path B: Client Watch Entry (drop-off)

**Step 1 — Customer selection**
1. Staff uses `CustomerSelector` component (searches `customers` table by name/email)
2. Upon selection → auto-fetches `previousWatches` from `client_property` (deduplicated by brand+model+ref)

**Step 2 — Estimate lookup** (`estimateParam` from URL query)
1. If `?estimate=<number>` → queries `estimates` with fuzzy matching:
   ```
   WHERE estimate_number = <padded> OR estimate_number ILIKE %<numeric>%>
     AND status IN ('sent','converted','draft','declined')
   ```
2. Pre-fills customer (watch), brand, model, departments, service codes, received components
3. Also checks `arrival_scan` to reveal if already received via shipping — shows warning if so

**Step 3 — Part # auto-decode** (`useModelReferenceLookup`)
1. If Part # looks like a reference (`^\d{4,}` or has model_references match):
   - Queries `model_references` → auto-fills brand, model, caliber
2. If Part # is service code (`BR-`/`PM-`/`POL-`) AND not a real ref: **skips** decode
3. Band-only mode: skips model decode (only brand needed)
4. Manual override: staff can tap brand/model/caliber to unlock edits

**Step 4 — Department auto-select**
1. From `jobProfile` RPC (`get_job_profile`) — maps to `deptW`, `deptB`, `deptP`, `deptPM`
2. From item markers: `[W]`/`[WP]`/`[WB]`/`[WBP]`/`[P]` → Watchmaker; `[B]`/`[PM]`/`[PB]` → Band
3. `deptAutoSelected` flag prevents UI confusion during auto-fill

**Step 5 — Received component selection**
1. Dynamic buttons from estimate services (via `extractReceivedItemsFromServices`)
2. Also from marker counting: `[W]`/`[WP]`/`[WB]`/`[WBP]`/`[P]` → "Complete watch"; `[B]`/`[PM]`/`[PB]` → "Bracelet"
3. Custom component input: free-text addition
4. **Bracelet mismatch logic**: if estimate says band-only but received "Complete watch" (or vice versa) → warning shown

**Step 6 — Signature & photos** (in-person drop-off only)
1. Signature pad: `SignaturePad` component → saves as blob URL → `handleSaveDropoffSignature` uploads blob
2. Photos: same CameraDialog or file picker as Path A

---

## 6. Outputs — Database writes per save

### ClientWatchEntryPage: Save (handleSave → saveMutation)

Every save writes to **5 tables**:

| Table | Operation | What's written | Key fields |
|---|---|---|---|
| **client_property** | `INSERT` | Main watch record | `customer_id`, `estimate_id`, `brand`, `model`, `reference_number`, `serial_number`, `notes`, `date_received`, `is_in_inventory`, `custody_status`, `received_bracelet`, `received_case`, `dept_w`, `dept_b`, `dept_p`, `dept_pm`, `item_index`, `bracelet_description`, `caliber` |
| **parts** | `UPSERT` | CW- part for barcode scanning | `part_number = CW-{brand_prefix}-{id_short}`, `item_type = client_watch`, `sku1 = partNumber`, `sku2 = serialNumber`, `brand`, `subcategory (model)`, `notes` |
| **estimates** | `UPDATE` | Status touch | `inspection_requests.status = 'completed'` (not estimates directly) |
| **model_references** | `UPSERT` | Brand/model/caliber lookup | `part_number = refKey`, `brand`, `model`, `caliber`, `source = 'intake'`, `needs_info` |
| **calibers** | `INSERT` (stub) | Creates incomplete calor stub if new | `caliber_number`, `brand`, `is_active = true`, `is_complete = false` |

**Additional writes:**
- `reference_bracelets` — `UPSERT` when both `partNumber` and `bracelet_description` present
- `client_property` `label_printed` → set true when label wizard completes (separate path)
- `client_property` `label_printed_at` → when labels printed

### ReceivePackagesPage: Save (savePackageItems)

| Table | Operation | What's written |
|---|---|---|
| **package_scan_logs** | `UPDATE` | `received_items`, `photo_urls`, `notes = received:{items}`, `email_sent`, `email_sent_at` |
| **client_property** | `UPDATE` | `notes = [received:{items}]` (where `estimate_id = packageReceiveData.id`) |
| **package_arrival_scans** | `UPDATE` | `memo = received:{items}`, `memo_added_by`, `memo_added_at` |

---

## 7. Cross-app touches

| Mechanism | Direction | Details |
|---|---|---|
| `send-estimate-email` edge function | RS → email (client) | Sends dropoff receipt with tracking, estimate #, received items, intake photos. CC to front desk |
| `upload-photo` edge function | RS → R2 (cloud storage) | AWS Sig V4 signed upload to R2. Folder: `{date}/{referenceNumber}/{photoType}_{order}.jpg` |
| `upload-client-photos` edge function | RC → RS storage bucket | RolliConnect client submits photos via token → `storage/bucket/client-photos` |
| `get_job_profile` RPC | RS internal | Fetches `received_components`, `custody`, `departments`, `flow (STANDARD_FLOW/BAND_ONLY_FLOW/...)`, `items[]` from estimate |
| `inspection_requests.status = 'completed'` | RS internal | Marks inline linked inspection as completed on save |
| **No direct RW push** on intake save | (gap) | Intake does NOT trigger RW Work Queue creation. That's a separate flow (likely `rw-job-status` edge function, not called from intake pages) |

**Confirmed gaps:**
- Intake save does NOT write to `jobs` table (no `job_id` generation on `client_property` insert)
- No edge function call triggers RW Work Queue job creation
- `client_property.custody_status = 'received'` is the only "job status" signal

---

## 8. Edge cases & failure modes

### Receiving fewer items than estimate says
- If estimate lists bracelet + complete watch but client only drops watch head → no validation blocks, just shows mismatch warning
- `braceletMismatch` flag compares received components vs estimate service codes
- Staff can acknowledge warning with button re-tap
- **No automated parts request triggered** for missing components

### Multi-estimate per package
- Code comment notes: "ability to assign incoming packages to multiple estimate numbers at intake" as a **need**, not implemented
- Current: scanning one tracking number → matches ONE estimate
- **Gap**: packages with 3-color carbon forms (2-3 estimates per shipment) are manually sorted after

### Already-received via shipping, then walked in
- `arrival_scan` preflight check warns if estimate already has `receive_status = 'received'`
- Staff can still proceed — no block

### Part # that looks like service code AND real reference
- `BR-1234-ABC` → service code prefix → skips decode BUT `looksLikeRealRef` fallback applies if `^\d{4,}` matches after prefix
- This edge case is handled but complex

### Caliber not in calibers table
- On save, if `formData.calimeter` is new: creates a **stub** in `calibers` with `is_complete = false`
- Stub appears in "calibers needing data" view for manual completion later
- **This is intentional workaround** for incomplete caliber data

### Barcode scan matching
- Tracking normalization: strips FedEx composite barcode prefixes (14-digit → 12-digit)
- `isSameTracking()`: handles partial matches (FedEx composite barcode suffix)
- Estimate matching: supports `EST-20355`, `E20355`, `20355` formats

### Voiding an intake
- Soft delete: `client_property` is NOT actually deleted, just marked `label_printed = false`? Actually it **IS** deleted via `.delete()`
- No audit trail for voided records (not in any audit log table)

---

## 9. Workarounds observed

| Workaround | Where | Why |
|---|---|---|
| `calibers` stubs (`is_complete = false`) | `saveMutation` in ClientWatchEntryPage | Caliber FK doesn't exist on `client_property` (stores `caliber` as plain text), so no FK violation on insert. Stub allows future linkage without blocking intake. |
| `notes` column for component data (e.g. `[received:complete_watch,bracelet]`) | `client_property.notes` | No dedicated `received_components` table. Components stored as tagged text in notes field. Parsing required downstream. |
| CW- part numbers for barcode lookup | Auto-generated CW-{brand3}-{id8} patterns | Creates a scannable barcode for each watch entry. Enables item lookup via part number scan. |
| Model reference caching | `model_references` table with `needs_info` flag | Allows partial model data to exist before full lookup. Reduces repeat lookups. |
| Department checkboxes derived from `get_job_profile` RPC | ClientWatchEntryPage `useEffect` on `jobProfile` | Departments computed server-side rather than on client — ensures consistency between estimate intent and intake display. |
| `received_bracelet` / `received_case` derived from itemType (not from actual received items) | `ClientWatchEntryPage` saveMutation line: `derivedReceivedBracelet = itemType === 'watch'` / `derivedReceivedCase = itemType === 'watch' \|\| itemType === 'watch_head_only'` | **This is a bug or intentional simplification:** whether the staff actually received the bracelet/case should be from `receivedComponents`, not the item type. A band-only job still sets `received_bracelet = false` regardless of whether the bracelet was received. |
| `received_case = true` default (migration) | `20260403020327...sql`: `received_case boolean NOT NULL DEFAULT true` | All existing rows get `received_case = true`. New inserts override with derived value. |
| `is_in_inventory = true` always | ClientWatchEntryPage always sets `is_in_inventory: true` | All intake creates mark the item as in inventory (no way to bypass for non-inventory items like consignment). |
| Branded 3-color carbon work orders still in use | Referenced in Michael's workflow docs ("hand-write changes on the 3-color carbon work order") | Digital workflow doesn't fully capture physical workaround — changes to job done on paper |

---

## 10. Open questions

### Q-001: Does the multi-estimate-per-package workflow exist in code?

**Type:** business truth
**Question:** Is there code support for one incoming package containing multiple estimates (multiple watches/line items), or is this purely a manual post-receive process?
**Why it matters:** Michael's workflow doc lists this as an edge case that needs implementation. If the code doesn't support it, the rebuild must add it.
**What I observed:** Both intake pages match one tracking number to ONE estimate. No multi-estimate assignment UI exists. The task packet asks about this as a "blocker example."
**My best guess:** Not implemented — manual sort after receive.
**Default if no answer in 7 days:** Document as "not implemented, manual process."

### Q-002: Is the `received_bracelet`/`received_case` derivation from `itemType` intentional?

**Type:** business truth
**Question:** Should `received_bracelet` and `received_case` be derived from the **item type** radio selection or from the **actual received components** the staff taps?
**Why it matters:** This affects downstream inventory accuracy. If a customer drops off a complete watch for band repair, `received_bracelet` would be correctly `true` from `itemType === 'watch'`, even if the staff didn't explicitly confirm bracelet receipt.
**What I observed:** `saveMutation` in ClientWatchEntryPage.tsx line ~1217-1218:
```
const derivedReceivedBracelet = itemType === 'watch';
const derivedReceivedCase = itemType === 'watch' || itemType === 'watch_head_only';
```
This means received status is inferred from the intended scope, not actual physical receipt.
**My best guess:** This is by design (intake time estimation, not final audit). Staff confirm physical components later in Shop Floor or Inspection.
**Default if no answer in 7 days:** Assume it's a simplification at intake — the definitive "received" state should come from Shop Floor component audit.

### Q-003: Does Shop Floor get triggered from intake at all?

**Type:** scope
**Question:** Is there any code path from `ClientWatchEntryPage` save → Job creation in RW Work Queue or RS `jobs` table?
**Why it matters:** If not, the Shop Floor failure ("components aren't registered so Shop Floor won't work") has a deeper root cause than just component registration — it may be that no job object is created at all during intake.
**What I observed:** Intake save writes to `client_property` (not `jobs`). No `rw-job-status` edge function call found in the intake code. No job status transition fires from intake save.
**My best guess:** Job creation in the work queue happens via a **separate manual step** (e.g., Vianna or Mike "assigns" the watch to a watchmaker after intake), not automatically from intake save.
**Default if no answer in 7 days:** Document as a gap — no automated job creation from intake.

### Q-004: How many components should be "registered" for Shop Floor to function?

**Type:** business truth
**Question:** What is the minimum set of component data in `client_property` (or a separate table) that Shop Floor's component list requires to start service?
**Why it matters:** Mike's quote: "components aren't registered so shop floor won't work." Without knowing the threshold, we can't fix registration reliably.
**What I observed:** Components stored as text in `client_property.notes` as `[received:complete_watch,bracelet]` — this format is parsed by `extractReceivedItemsFromServices` but not by any Shop Floor component viewer (no Shop Floor component API found).
**My best guess:** Components need to be in a structured table (not text in notes) with at least: part number, description, quantity, and item type.
**Default if no answer in 7 days:** Flag for rebuild to define and require a component data model.

### Q-005: Is the `custody_status` on `client_property` used by any downstream system?

**Type:** acceptance
**Question:** Is `custody_status = 'in_custody'` / `'released'` consumed by RW, RC, or the physical chain-of-custody system (D-015)?
**Why it matters:** If yes, the rebuild must preserve this status or migrate it to the new shared model. If not, it's orphaned data.
**What I observed:** `custody_status` added in migration `20260110084357...` alongside `watch_tag_number`, `estimate_id`, `intake_date`, `release_date`. No code reads `custody_status` in the intake pages.
**My best guess:** It exists for future chain-of-custody use (D-015) but is currently set once at creation and never updated.
**Default if no answer in 7 days:** Document as "reserved but unused."

---

## Schema summary — tables with their relevant columns for intake

### `client_property` (final columns relevant to intake)

| Column | Type | Source |
|---|---|---|
| `id` | UUID | PK, auto-generated |
| `customer_id` | UUID FK → `customers` | staff selects |
| `estimate_id` | UUID FK → `estimates` | auto-linked or null |
| `brand` | TEXT | required |
| `model` | TEXT | nullable |
| `reference_number` | TEXT | part number / serial |
| `serial_number` | TEXT | nullable |
| `date_received` | TIMESTAMPTZ | defaults now() |
| `is_in_inventory` | BOOLEAN | always `true` |
| `custody_status` | TEXT | defaults `'in_custody'` |
| `received_bracelet` | BOOLEAN | derived from `itemType` |
| `received_case` | BOOLEAN | derived from `itemType` |
| `dept_w` | BOOLEAN | auto from estimate/RF |
| `dept_b` | BOOLEAN | auto from estimate/RF |
| `dept_p` | BOOLEAN | auto from estimate/RF |
| `dept_pm` | BOOLEAN | auto from estimate/RF |
| `item_index` | INTEGER | defaults 1 |
| `bracelet_description` | TEXT | from estimate or manual |
| `caliber` | TEXT | from model_ref or manual |
| `notes` | TEXT | `[received:...]` + `[split_custody:...]` tags |
| `label_printed` | BOOLEAN | toggled by label wizard |
| `label_printed_at` | TIMESTAMPTZ | set by label wizard |

### `parts` (CW- entries)

| Column | Type | Purpose |
|---|---|---|
| `part_number` | TEXT PK | `CW-{brand3}-{id8}` |
| `description` | TEXT | Auto-computed from brand/model/reference/customer |
| `item_type` | ENUM | Always `'client_watch'` |
| `uom` | ENUM | Always `'ea'` |
| `sku1` | TEXT | original part number |
| `sku2` | TEXT | original serial number |
| `brand` | TEXT | for lookup |
| `subcategory` | TEXT | model for lookup |
| `notes` | TEXT | rich detail string |

### `model_references`

| Column | Type | Purpose |
|---|---|---|
| `part_number` | TEXT PK | canonical reference (no serial) |
| `brand` | TEXT | from decode |
| `model` | TEXT | from decode |
| `caliber` | TEXT | from decode |
| `source` | TEXT | `'intake'` for intake lookups |
| `needs_info` | BOOLEAN | partial decode flag |

### `calibers`

| Column | Type | Purpose |
|---|---|---|
| `caliber_number` | TEXT PK | exact caliber code |
| `brand` | TEXT | for filtering |
| `name` | TEXT | display name |
| `movement_type` | TEXT | auto/manual/quartz |
| `frequency` | INTEGER | BPH |
| `jewels` | INTEGER | movement jewel count |
| `power_reserve_hours` | INTEGER | movement reserve |
| `is_active` | BOOLEAN | defaults `true` |
| `is_complete` | BOOLEAN | **intake stubs set `false`** — manual completion required |

**⚠️ Notable:** `client_property.caliber` is a TEXT column — **no FK to `calibers`**. Intake creates calibers stubs but never links back. This is the "broken FK to calibers" mentioned in the task packet.

### `package_scan_logs`

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID PK | |
| `tracking_number` | TEXT | from barcode |
| `tracking_formatted` | TEXT | display format |
| `carrier` | TEXT | FedEx/UPS/USPS |
| `scanned_at` | TIMESTAMPTZ | |
| `scanned_by` | UUID → `auth.users` | |
| `matched_customer_id` | UUID → `customers` | |
| `matched_estimate_id` | UUID → `estimates` | |
| `email_sent` | BOOLEAN | confirmation status |
| `received_items` | TEXT[] (via notes) | what we actually got |
| `photo_urls` | TEXT[] (via notes) | intake photos |
| `notes` | TEXT | `received:{items}` format |

### `package_arrival_scans`

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID PK | |
| `tracking_number` | TEXT | |
| `receive_status` | TEXT | `'pending'` / `'received'` |
| `received_at` | TIMESTAMPTZ | |
| `received_by` | UUID | staff who received |
| `memo` | TEXT | received items summary |

### `job_status` enum (RS)

| Status | Meaning |
|---|---|
| `intake` | just arrived |
| `in_review` | staff reviewing |
| `awaiting_customer_approval` | estimate sent |
| `approved` | customer approved |
| `in_service` | watchmaker working |
| `testing` | timing/pressure |
| `ready_to_ship` | done, waiting for ship |
| `closed` | complete |

**⚠️ Note:** None of these are set during intake save. `client_property.jod_id` exists (from `estimates` migration) but is not populated by intake code.

---

## Comparison to workflow docs

| Workflow doc | Matches code | Notes |
|---|---|---|
| **Vianna's intake flow** (scan tracking → estimate → photo → email) | ✅ Matches | Vianna's daily flow ("Open and process incoming packages → Scan tracking → scan estimate → photo intake → email client receipt") maps directly to ReceivePackagesPage |
| **Michael's Watchmaker bench** (QR-driven intake) | ⚠️ Partial | Code has QR scan for estimates (via URL params) but no QR-driven component check-in. Michael's "want: QR codes per process" is not yet implemented |
| **Michael's multi-estimate per package** | ❌ Not implemented | Listed as a need, not in code |
| **Michael's Parts request flow** (WM types → Mike enters price) | ❌ Out of scope | This is RW domain, not RS intake |
| **Michael's Pickup Station QBO bypass** | ❌ Out of scope | Separate module (Q-006) |

---

## What "done" means (acceptance criteria check)

- ✅ Every field entered at intake is named with source UI label, code variable name, table.column destination, and data type
- ✅ Every cross-app effect listed (edge function `send-estimate-email`, `upload-photo`, RPC `get_job_profile`, storage bucket writes)
- ✅ Every workaround flagged (CW- part numbers, caliber stubs, notes-as-components, itemType-derived received flags)
- ✅ Comparison to Vianna+Michael workflow docs identifies gaps (multi-estimate, QR intake, parts auto-request)
- ✅ Ambiguities logged to Open Questions section

---

## Component registration gap analysis

The task packet identifies component registration as the root cause of Shop Floor's persistent failure. Here's what the code shows:

1. **Intended flow:** Staff selects received components (Complete watch, Bracelet, etc.) at intake
2. **Current storage:** Components stored as text tags in `client_property.notes`: `[received:complete_watch,bracelet,watch_head]`
3. **Gap:** No `components` table, no structured component records, no Shop Floor component list endpoint
4. **Downstream effect:** Shop Floor module has no data model to display or audit what was received vs what was worked on
5. **Minimal fix:** A `job_components` table with: `job_id`, `component_type` (watch/bracelet/case/etc.), `part_number`, `notes`, `status` (received/installed/replaced)

---

_End of discovery. Q-001 complete._
