# Q-009 — Map inspection photo storage + R2 access patterns

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-29
**Human answers applied:** D-019 (Stage 1 possession photos), W-38 (before/after at pickup), D-008/ROLLITIME-INTEGRATION §6 (dial library), A-20260628-014 (intake photos at pickup)
**Inputs read:**
- `apps/rs/supabase/functions/upload-photo/index.ts`, `get-photo/index.ts`, `upload-client-photos/index.ts`
- `apps/rs/src/pages/inspection/InspectionCapturePage.tsx`, `InspectionLibraryPage.tsx`, `InspectionDetailPage.tsx`
- `apps/rs/src/pages/sales/PickupStationPage.tsx` (intake photo grid)
- `apps/rs/src/pages/intake/ReceivePackagesPage.tsx` (intake upload paths)
- `apps/rw/supabase/functions/ai-assistant/index.ts` — `toolGetInspectionPhotos`
- `documentation/ROLLITIME-INTEGRATION.md` §6
- `documentation/discovery/Q-006-pickup-station.md`
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**Photo storage & access** — Documents where watch photos live (R2, Supabase Storage, RT buckets), who writes and reads them, galleria organization today, and changes needed for: shared dial-photo library (D-008 + RT), D-019 two-stage capture, W-38 pickup before/after comparison, and RW read access.

Vianna's pain: trade clients with **30–50 watches × 10–40 photos** — galleria unwieldy (W-48).

---

## 2. Trigger

| Event | Photo type | Writer |
|-------|------------|--------|
| Package receive (Stage 1) | Arrival photos | RS `ReceivePackagesPage` → Supabase Storage |
| Drop-off intake (Stage 1) | Possession photos | RS `ClientWatchEntryPage` / receive flows |
| Inspection capture | Through-crystal Ipevo + microscope | RS `InspectionCapturePage` → R2 |
| Client portal upload | Client-submitted | RS `upload-client-photos` → `client-photos` bucket |
| Pickup exit | Release photos | RS `PickupStationPage` → `pickup_sessions` |
| Ship exit | Package photos | RS `SalesOrderFulfillPage` → `shipment_logs` |
| RT testing station | Dial stills + Witschi | RT → `watch-stills`, `witschi-photos` (separate DB) |

---

## 3. Actors

| Actor | Read | Write |
|-------|------|-------|
| RS inspection staff | R2 via `get-photo` proxy | `upload-photo` |
| RS pickup staff | Intake refs + exit capture | Pickup session URLs |
| RW watchmakers | **Blocked in UI**; AI assistant tool only | None (W-26 wish) |
| RC clients | Portal upload token | `upload-client-photos` |
| RT operators | RT buckets | RT capture routes |
| Future Authenticator | RT dial library (read) | None |

---

## 4. Photo silos today (three systems)

| Silo | Storage | Key path pattern | DB index |
|------|---------|------------------|----------|
| **Inspection (RS)** | Cloudflare R2 bucket `inspections-photos` | `{date}/{referenceNumber}/{photoType}_{photoOrder}.jpg` | `inspection_photos.photo_url` |
| **Intake (RS)** | Supabase `attachments` / `shipment-photos` | `package-photos/…`, `dropoff/{estimate}/…` | `package_scan_logs.photo_urls` TEXT[] |
| **Client upload (RS)** | Supabase `client-photos` | `{requestId}/{uuid}.ext` | `client_photo_submissions` |
| **Dial library (RT)** | RT Supabase `watch-stills`, `witschi-photos` | per `reading.*_photo_url` | `reading` table |
| **Pickup exit (RS)** | Supabase (URLs in session) | varies | `pickup_sessions.pickup_photo_urls` |

**No unified index** across silos — W-38 before/after requires cross-silo read at pickup.

---

## 5. R2 inspection bucket (detail)

### Upload path (`upload-photo`)

```json
{
  "inspectionId": "uuid",
  "referenceNumber": "116610LN",
  "photoData": "base64…",
  "photoType": "webcam" | "microscope",
  "photoOrder": 1,
  "date": "2026-06-29"
}
```

**Object key:** `{date}/{referenceNumber}/{photoType}_{photoOrder}.jpg`

**Env:** `R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, `R2_ENDPOINT`, `R2_BUCKET_NAME`

**DB:** `inspections` parent row; `inspection_photos` child with full R2 URL in `photo_url`

### Read path (`get-photo`)

`GET /functions/v1/get-photo?path={url-encoded relative path}&apikey={anon}`

- `verify_jwt = false` in `config.toml` — access gated by anon key + path knowledge
- UI: `getProxiedUrl()` hardcodes bucket name `inspections-photos`

### Capture UI

| Page | Route | Role |
|------|-------|------|
| `InspectionCapturePage.tsx` | `/inspection/capture` | Create inspection + upload |
| `InspectionDetailPage.tsx` | `/inspection/:id` | View session |
| `InspectionLibraryPage.tsx` | `/inspection/library` | Galleria browse |
| `InspectionImportPage.tsx` | `/inspection/import` | ZIP bulk import |

---

## 6. Intake photos (Stage 1 — D-019)

| Source | Upload mechanism | Stored |
|--------|------------------|--------|
| Package receive | Direct Supabase Storage upload | `package_scan_logs.photo_urls` public URLs |
| Drop-off | `dropoff/{estimate_number}/{timestamp}.ext` | same table |
| Some intake flows | `upload-photo` edge (per Q-001) | Mixed |

**Pickup reference grid** (`PickupStationPage.tsx`): reads `package_scan_logs` for estimate; resolves URLs — **bucket inconsistency risk** (`attachments` vs `shipment-photos`).

**W-38 gap:** Drop-off photos on `ClientWatchEntryPage` may **not** appear in pickup grid today (Q-006 open question — **answered A-014: yes, must**). Rebuild must unify Stage 1 photo index: `shared.intake_photos` with `stage: possession`, `source: dropoff | shipping_receive`.

---

## 7. Galleria organization today

| Dimension | Current behavior |
|-----------|------------------|
| Primary grouping | Per `inspections` session (date + reference) |
| Customer view | `CustomerDetailPage` / `JobHistoryDialog` — lists inspections chronologically |
| Trade account scale | All inspections for client — **no date folders** (W-48 pain) |
| Per-watch | Filtered by `reference_number` on inspection row |
| Per-job | Via estimate → client_property → inspections chain |

**Not date-folder organized** despite thousands of photos per heavy client.

---

## 8. Why RW can't see photos today

| Layer | Gap |
|-------|-----|
| **UI** | No RW photo viewer page (W-26) |
| **Auth** | RW has no R2 credentials; must proxy RS `get-photo` |
| **Data** | RW resolves chain: `estimate_number → estimates → client_property → inspections → inspection_photos` |
| **Implementation** | Only `ai-assistant/index.ts` `toolGetInspectionPhotos()` — not shop floor |

**This is both auth and UI gap** — proxy path exists for AI tool only.

---

## 9. W-38 — Before/after at pickup

### Required comparison

| Side | Source (rebuild) | Today |
|------|------------------|-------|
| **Before** | `shared.intake_photos` Stage 1 (drop-off + shipping) | `package_scan_logs` only (partial) |
| **After** | `pickup_sessions.pickup_photo_urls` | Exists |

### Proposed UI contract

```
PickupStationPage:
  leftColumn: intake_photos[] filtered by estimate + asset_id
  rightColumn: pickup_capture_photos[]
  operator: must acknowledge comparison checkbox before complete
```

### Index query (rebuild)

```sql
-- shared.intake_photos (proposed)
estimate_id, client_property_id, stage ('possession'|'verification'),
source ('dropoff'|'shipping_receive'|'inspection'|'rt_dial'),
storage_bucket, storage_path, captured_at
```

Pickup reads `stage = possession` for before; exit writes `stage = release` or uses `pickup_sessions`.

---

## 10. RolliTime dial-photo library integration

Per ROLLITIME-INTEGRATION §6 + Q-008:

| Aspect | RS inspection R2 | RT dial library |
|--------|------------------|-----------------|
| Owner | RS | RT |
| Content | Through-crystal inspection | Dial-removed testing stills |
| Key | `reference_number` + date folder | `ref_number` + `serial` |
| RS access | Native | **Metadata index** + signed URL proxy (`get-rt-photo`) |
| Authenticator | Future read via RT API | Primary consumer |

**Do not merge buckets** — different capture context; unified **client file UI** composes both sources.

---

## 11. Proposed galleria reorganization (minimal invasive)

Without rebuilding entire UI:

| Change | Effort | Effect |
|--------|--------|--------|
| Add `folder_date` computed column on `inspections` | Migration | Sort/filter by month |
| `InspectionLibraryPage` date accordion | UI | W-48 date folders |
| Trade account filter: `customer_id` + `reference_number` | UI | Narrow 1000+ photo sets |
| Virtual folder: `{customer}/{YYYY-MM}/{ref}/` | Display layer | No blob migration |

**Retention:** unchanged — R2 paths stable; index-only reorganization.

---

## 12. RW read access — specific changes

| # | Change | Repo |
|---|--------|------|
| 1 | RS edge `get-inspection-photos-for-estimate?estimate=` | RS |
| 2 | Bearer `ROLLIWORKING_API_KEY` auth on above | RS |
| 3 | RW `useInspectionPhotos(estimateNumber)` hook | RW |
| 4 | Inspection tab on Work Queue job detail | RW UI (W-26) |
| 5 | Log every proxy read to `integration_event_log` | D-020 |

No R2 credentials on RW — all via RS proxy.

---

## 13. D-019 two-stage photo discipline

| Stage | Photos | Table (rebuild) |
|-------|--------|-----------------|
| Stage 1 possession | Drop-off, shipping receive | `shared.intake_photos` |
| Stage 2 verification | Comparison at Receive Watch | Link to Stage 1; discrepancy log |
| Inspection | Through-crystal | `inspection_photos` (R2) — post-intake |
| RT testing | Dial removed | RT buckets + RS index |
| Pickup release | Before/after (W-38) | intake_photos + pickup_sessions |

W-36 AI scaffolding (`ai_detected_items` on intake photos) attaches to Stage 1 records per WISHLIST.

---

## 14. Edge cases

| Case | Issue |
|------|-------|
| `get-photo` anon key | Path enumeration risk — rebuild should use short-lived signed tokens |
| Mixed bucket URLs in `package_scan_logs` | Pickup grid broken for some intake paths |
| Duplicate `reference_number` inspections | Galleria shows multiple sessions — correct but noisy |
| RT photos not on RS file | Client file incomplete until Q-008 index live |

---

## 15. Open questions

### Q-009-A: Migrate historical intake photos to R2 or keep Supabase Storage?

**Type:** infrastructure
**Default:** Keep Stage 1 on Supabase; unify via index table only (cheaper migration).

### Q-009-B: Should pickup before/after require paired photo counts (1:1)?

**Type:** UX policy
**Default:** Soft warning if before count > 0 and after count = 0; hard block per W-38 only on operator ack checkbox.

---

## 16. Minimum invasive recommendations summary

1. **`shared.intake_photos`** index unifying drop-off + shipping (D-019, W-38)
2. **Pickup grid** reads index — not raw `package_scan_logs` only
3. **Galleria date accordion** on `InspectionLibraryPage` (W-48)
4. **RS proxy endpoint** for RW inspection photo read (W-26)
5. **RT dial index** on RS client file via Q-008 — separate from R2 inspection corpus
6. **D-020** logging on every cross-app photo proxy

---

## 17. Acceptance criteria check

- ✅ R2 path structure documented
- ✅ Upload/read paths per function named
- ✅ Auth model documented (anon key proxy today; signed URL rebuild)
- ✅ RW read gap named with specific fix list
- ✅ Galleria reorganization proposed without blob migration
- ✅ RT integration plan distinct from RS R2
- ✅ W-38 before/after addressed in proposed organization

---

_End of discovery. Q-009 complete._
