# Q-015 — Map intake workflow variants

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-29
**Depends on:** Q-001
**Human answers applied:** A-20260628-001 (multi-estimate), A-20260628-002 (components not itemType), D-019 (two-stage), W-31 (multi-item numbering), W-32 (related accounts), W-36 (backfill no-estimate)
**Inputs read:**
- `apps/rs/src/pages/intake/ReceivePackagesPage.tsx`
- `apps/rs/src/pages/intake/ClientWatchEntryPage.tsx`
- `apps/rs/src/pages/estimates/CreateEstimatePage.tsx`
- `documentation/discovery/Q-001-receive-watch-module.md`
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**Intake variants** — Document how estimate-first, no-estimate, multi-estimate-per-label, and husband/wife shipments map to code — and how each should map to D-019 Stage 1 (possession) vs Stage 2 (verification + commit).

---

## 2. Variant matrix

| Variant | Frequency | Stage 1 (ReceivePackages) | Stage 2 (ClientWatchEntry) | D-019 compliant? |
|---------|-----------|---------------------------|----------------------------|------------------|
| **(a) Estimate-first** | ~99% | ✅ Match, photo, memo | ✅ Commit inventory | ⚠️ Stages optional |
| **(b) No estimate** | Recurring | ⚠️ `needs_review` / customer-only | ❌ Estimate required to save | ❌ |
| **(c) Multi-estimate / label** | Recurring | ❌ One estimate per scan row | ❌ One estimate per session | ❌ |
| **(d) Husband/wife one tracking** | Recurring | ❌ One customer per package | ❌ | ❌ |
| Multi-item, one estimate | Common | ✅ Marker counts | ✅ `item_index` UI | ⚠️ No per-item possession FK |

---

## 3. Variant (a) — Estimate-first standard

**Flow:** Tracking or EST# scan → package received → photos/email → Receive Watch → `client_property` + labels → RW push.

**Code:** `searchEstimateByTracking()`, `savePackageItems()`, `ClientWatchEntryPage.saveMutation`.

**Gaps:** Stage 2 not gated on Stage 1 completion; components in notes not structured (Q-011).

---

## 4. Variant (b) — No estimate / backfill (W-36)

**Operational:** Vianna sets aside; Mike creates estimate on `/estimates/new`; staff re-links tracking.

**Code support:**
- ✅ Unmatched tracking → `needs_review` or customer-only match
- ✅ Possession save without estimate (`packageReceiveData.id` null)
- ❌ `ClientWatchEntryPage` **requires** `estimateNumber` to save (~L1612)
- ❌ Drop-off tab requires estimate found

**Minimal fix:** Allow Stage 2 save with `estimate_id: null` + `backfill_status: pending_estimate`; Hit List queue; deep-link `/estimates/new?tracking=…`.

---

## 5. Variant (c) — Multi-estimate per package (A-001)

**Patterns:** One client multiple jobs one tracking; two clients one shipment.

**Code constraints:**
- `matched_estimate_id` — single UUID per `package_scan_logs` row
- `searchEstimateByTracking()` — `.limit(1).maybeSingle()`
- `handleEstimateMatch()` overwrites one estimate's `tracking_number`

**Not the same as:** Multi-**item** per single estimate (`jobProfile.items[]`, "Item 2 of 3") — that works today.

**Minimal fix:** Junction `package_estimate_links(scan_id, estimate_id)`; UI "+ Add estimate to this package"; stop single-tracking overwrite.

---

## 6. Variant (d) — Husband/wife (W-32)

**Operational:** Two `customers`, two `estimates`, one `tracking_number`.

**Code:** No `related_account_id` / household model; one customer per `handleCustomerMatch()`.

**Minimal fix:** Build on (c) + `customers.household_id`; package card shows both clients; **do not merge** RC clients — link at package level only.

---

## 7. D-019 mapping per variant

| Stage | Estimate-first | No-estimate | Multi-estimate | Spouses |
|-------|----------------|-------------|----------------|---------|
| **Stage 1** | `package_arrival_scans` + photos | Same; `needs_review` | One possession record, **many** estimate links | Same + household display |
| **Stage 2** | Verify + `client_property` per asset | Blocked until estimate OR allow pending | **Per estimate** verification pass | Per spouse estimate |
| **Audit** | Compare Stage 1 memo vs Stage 2 components | Flag long pending | Per-estimate discrepancy | Per-client attribution |

---

## 8. W-31 multi-item auto-numbering

Today: sequential `item_index` under one estimate in ClientWatchEntry.

Rebuild: "Add another item" → `1 of 2`, `2 of 2` identifiers; each item own possession + verification records (A-002).

---

## 9. Estimate creation entry points (none in intake)

| Path | Intake tie-in |
|------|---------------|
| `/estimates/new` | External to intake |
| `/intake/leads` → new estimate | Lead prefill only |
| `handleEstimateMatch` | Links existing estimate — does not create |

Backfill is always **two-step:** possession → estimate elsewhere → link.

---

## 10. Open questions

### Q-015-A: Should no-estimate possession auto-expire or escalate after N days?

**Type:** operational
**Default:** 7-day stagnation tracker (W-31 table) applies to possession-without-estimate rows.

---

## 11. Acceptance criteria check

- ✅ Four variants documented with code evidence
- ✅ D-019 Stage 1/2 mapping per variant
- ✅ Minimal additions proposed per gap
- ✅ W-31, W-32 applied

---

_End of discovery. Q-015 complete._
