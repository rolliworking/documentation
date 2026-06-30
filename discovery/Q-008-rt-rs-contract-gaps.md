# Q-008 — Close RolliTime ↔ RS contract gaps

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-29
**Prior audit:** 2026-06-07 Qwen (intent-only contract)
**Human answers applied:** D-019 (two-stage intake), D-020 (silent failure banned, observable async status)
**Inputs read:**
- `documentation/ROLLITIME-INTEGRATION.md` (full)
- `apps/rt/documentation/SCHEMA.md`, `SPEC.md`, `workflow-batch-testing.md`
- `apps/rs/supabase/functions/job-status-update/index.ts`
- `apps/rw/supabase/functions/rollisuite-webhook/index.ts`, `rollisuite-status-push/index.ts`
- `apps/rs/src/pages/inspection/` (RT photo surfacing target)
- `documentation/workflows/MICHAEL-WORKFLOW.md` (testing-complete email)
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**RolliTime cross-app integration** — Defines how watch accuracy testing data and dial-photo library flow between RolliTime (RT), RolliSuite (RS), and RolliWorking (RW). RT is in active build; RS/RW are production. The 2026-06-07 audit correctly flagged the contract as **intent-only**. This document closes that gap with **proposed payload schemas**, endpoints, auth, and D-020 observability requirements.

**Universal join key:** `ref_number` + `serial` (barcode `ref#-serial`), mapped to RS `client_property` and RW `jobs` via estimate linkage.

---

## 2. Current state summary

| Flow | Contract in ROLLITIME-INTEGRATION.md | Implemented in code |
|------|--------------------------------------|---------------------|
| RT → RS | §4.1 test results + §6 photos | **No** |
| RS → RT | §4.2 caliber/reference pull | **No** (CSV seed only in RT docs) |
| RT → RW | §3.1 `in_testing` status push | **No** |
| RT ← RW | Not defined | **No** |

**Analog patterns to copy:** RW→RS `job-status-update` (Bearer `ROLLISUITE_API_KEY`); RS→RW `rollisuite-webhook` (`STATUS_UPDATE`).

---

## 3. Flow 1 — RT → RS (test results + dial-photo metadata)

### 3.1 Purpose

Land timing/PR results on the client file; expose dial-photo library metadata for inspection UI and future Authenticator.

### 3.2 Proposed endpoints

| Endpoint | Method | Owner |
|----------|--------|-------|
| `POST /functions/v1/rt-test-results` | Push completed test batch summary | RS edge (new) |
| `GET /functions/v1/rt-dial-photos?ref={ref}&serial={serial}` | Proxy read from RT library | RS edge (new) or RT-native with RS service token |

### 3.3 RT → RS test results payload

```json
{
  "event_id": "uuid",
  "event_type": "test_batch.completed",
  "occurred_at": "2026-06-29T14:30:00Z",
  "idempotency_key": "rt-batch-{batch_id}",
  "watch": {
    "ref_number": "116610LN",
    "serial": "A12345",
    "caliber": "3135"
  },
  "estimate_number": "EST-25252",
  "batch_id": "uuid",
  "segments": [
    {
      "position": 1,
      "dial_time_entered": "2026-06-29T12:00:00Z",
      "readings": [
        {
          "reading_id": "uuid",
          "drift_seconds": 2.1,
          "rate_s_per_day": 1.8,
          "amplitude_deg": 285,
          "beat_error_ms": 0.3,
          "power_reserve_hours": 48.2,
          "pass": true,
          "tolerance_profile_id": "uuid",
          "still_photo_path": "watch-stills/2026-06-29/116610LN-A12345/pos1_still.jpg",
          "witschi_photo_path": "witschi-photos/2026-06-29/116610LN-A12345/pos1_sheet.jpg"
        }
      ]
    }
  ],
  "summary": {
    "overall_pass": true,
    "max_amplitude_deg": 290,
    "min_power_reserve_hours": 47.5
  }
}
```

### 3.4 RS destination tables (rebuild target)

| RT field | RS table.column | Notes |
|----------|-----------------|-------|
| `watch.ref_number` + `serial` | `client_property.reference_number`, `serial_number` | Join key |
| `estimate_number` | `estimates.estimate_number` | Optional FK |
| Segment readings | `shared.timing_test_results` (new) or extend legacy `timing_tests` | D-020: separate from pre-RT RS tables |
| Photo paths | `shared.dial_photo_index` (metadata only; blobs stay in RT) | RS stores pointers, not copies |
| `event_id` | `integration_event_log` | D-020 audit BEFORE processing |

### 3.5 Dial-photo metadata API response (RS proxy)

```json
{
  "ref_number": "116610LN",
  "serial": "A12345",
  "caliber": "3135",
  "photos": [
    {
      "photo_id": "uuid",
      "capture_timestamp": "2026-06-29T12:05:00Z",
      "entered_dial_time": "2026-06-29T12:00:00Z",
      "section": "dial_up",
      "orientation": "12_up",
      "cycle": 1,
      "context": "section_capture",
      "thumbnail_url": "https://rt.../signed/...",
      "full_url": "https://rt.../signed/..."
    }
  ]
}
```

### 3.6 Auth

- RT → RS: `Authorization: Bearer {ROLLITIME_API_KEY}` (new secret, parallel to `ROLLISUITE_API_KEY`)
- RS validates key; writes `integration_event_log` with `status: processing` **before** INSERT
- On failure: `status: failed`, `error_message`; return **non-200** (D-020)

### 3.7 D-019 applicability

Test results attach at **Stage 2+** (post-verification). RT scan at testing station implies watch is already in shop possession — RT should reject push if RS reports no verified `client_property` row for ref-serial (optional gate, rebuild).

---

## 4. Flow 2 — RS → RT (caliber & reference data)

### 4.1 Purpose

RS authoritative reference library per §4.2; replaces CSV drift.

### 4.2 Proposed endpoints

| Endpoint | Method | Pattern |
|----------|--------|---------|
| `GET /functions/v1/reference-lookup?ref={ref}` | Single ref | RT poll on barcode scan |
| `GET /functions/v1/reference-sync?since={iso}` | Delta since timestamp | RT cron every 15m |

### 4.3 Reference record payload

```json
{
  "ref_number": "116610LN",
  "brand": "Rolex",
  "model_name": "Submariner Date",
  "caliber": "3135",
  "beats_per_hour": 28800,
  "tolerances": {
    "rate_s_per_day_max": 2.0,
    "amplitude_deg_min": 270,
    "power_reserve_hours_min": 48
  },
  "updated_at": "2026-06-28T10:00:00Z",
  "source": "rs_authoritative"
}
```

### 4.4 Pull mechanism recommendation

**Hybrid:** Realtime optional for hot refs; **required** delta poll cron on RT with `integration_sync_runs` table (D-020). RT stores local cache `reference_cache` keyed by `ref_number`.

### 4.5 Auth

- RT → RS: `Authorization: Bearer {ROLLITIME_API_KEY}` (read-only scope)
- RS returns 404 for unknown ref — RT surfaces toast per W-28

---

## 5. Flow 3 — RT → RW (status push: in_testing)

### 5.1 Purpose

When watch scanned at RT photo/testing station, RW sets `jobs.status = in_testing`.

### 5.2 Proposed endpoint

`POST /functions/v1/rt-status-push` on **RW** (new), mirroring `rollisuite-webhook`.

### 5.3 Payload

```json
{
  "event_id": "uuid",
  "event_type": "watch.entered_testing",
  "occurred_at": "2026-06-29T15:00:00Z",
  "idempotency_key": "rt-testing-{ref}-{serial}-{occurred_at}",
  "watch": {
    "ref_number": "116610LN",
    "serial": "A12345"
  },
  "estimate_number": "EST-25252",
  "rt_batch_id": "uuid",
  "confirmed": true
}
```

`confirmed: true` required per §3.1 note — deliberate operator confirm at RT station prevents mis-scan.

### 5.4 Keying mechanism

1. Primary: `estimate_number` → RW `jobs.estimate_number` (same as `rollisuite-webhook` `jobId`)
2. Fallback: `ref_number` + `serial` → RW `jobs` lookup
3. On ambiguity (multiple open jobs): return **409** with `candidate_job_ids[]` — do not silent-pick (D-020)

### 5.5 RW handler

Call existing `set_job_status(job_id, 'in_testing')` RPC; log to `job_status_logs`; return:

```json
{ "ok": true, "job_id": "uuid", "previous_status": "in_progress", "new_status": "in_testing" }
```

### 5.6 Auth

`Authorization: Bearer {ROLLITIME_API_KEY}` on RW edge function.

### 5.7 D-020 observability

RW writes `integration_event_log` row **before** RPC; UPDATE to `success`/`failed` with row count check.

---

## 6. Flow 4 — RT ← RW (reverse path)

### 6.1 Is it needed?

**Yes — limited scope.** Scenarios:

| Scenario | RT action |
|----------|-----------|
| Job cancelled while on RT bench | Invalidate in-progress batch; mark readings `voided` |
| Watch removed from testing early (`downgrade` scan) | Close RT session without pass |
| RW `in_testing` → `in_progress` exit | RT batch may complete normally |

### 6.2 Proposed endpoint

`POST /functions/v1/rw-testing-cancel` on **RT** (new).

### 6.3 Payload

```json
{
  "event_id": "uuid",
  "event_type": "testing.cancelled",
  "occurred_at": "2026-06-29T16:00:00Z",
  "estimate_number": "EST-25252",
  "watch": { "ref_number": "116610LN", "serial": "A12345" },
  "reason": "job_cancelled",
  "rw_job_id": "uuid",
  "rw_previous_status": "in_testing"
}
```

### 6.4 RT handler

- Find open `test_batch` for watch; set `status: cancelled`
- Do **not** delete readings (retention §6.3); mark `voided: true`
- Return 200 only if batch state updated OR already terminal; else 404/409

### 6.5 Auth

`Authorization: Bearer {ROLLIWORKING_API_KEY}` on RT.

### 6.6 Trigger

RW `set_job_status` side effect when transition **from** `in_testing` **to** `cancelled` | `in_queue` | `waiting_components` — webhook fan-out (new hook in RW, not poll).

---

## 7. RT cancelled mid-test scenario

| Step | System | Action |
|------|--------|--------|
| 1 | Staff scans downgrade at RW | `set_job_status` → `in_progress` |
| 2 | RW webhook | POST `rw-testing-cancel` to RT |
| 3 | RT | Voids open batch; UI shows "session ended" |
| 4 | RT | Does **not** push partial results to RS unless operator confirms save |
| 5 | RS | No client-file update until explicit `test_batch.completed` |

**D-020:** Each step logs observable status; partial RT→RS push prohibited without `event_type: test_batch.completed`.

---

## 8. Photo storage destination decision

| Corpus | Owner | Storage | RS relationship |
|--------|-------|---------|-----------------|
| Dial stills / Witschi sheets | **RT** | RT Supabase buckets `watch-stills`, `witschi-photos` | RS stores **metadata index** + signed URL proxy |
| Through-crystal inspection | **RS** | Cloudflare R2 `inspections-photos` | Unchanged (Q-009) |
| Stage 1 intake | **RS** | Supabase `attachments` | D-019 possession photos |

**Recommendation:** RT **owns** dial library blobs; RS never copies to R2. RS `get-photo` pattern extended as `get-rt-photo` proxy with TTL-signed URLs.

---

## 9. Auth pattern summary (cross-database writes)

| Caller | Callee | Secret | Scope |
|--------|--------|--------|-------|
| RT | RS | `ROLLITIME_API_KEY` | write test results, read reference |
| RT | RW | `ROLLITIME_API_KEY` | push in_testing |
| RW | RT | `ROLLIWORKING_API_KEY` | cancel testing session |
| RS | RT | `RS_RT_READ_KEY` | optional poll fallback |

All keys rotatable; stored in Supabase secrets per project. **No shared database.**

---

## 10. Gap closure matrix (2026-06-07 audit)

| Audit gap | Resolution in this doc |
|-----------|------------------------|
| No RT→RS payload | §3.3 schema + `rt-test-results` endpoint |
| No RS→RT pull | §4.3 reference payload + delta sync |
| No RT→RW push | §5.3 status payload + keying rules |
| No reverse path | §6 RW→RT cancel contract |
| Photo destination open | §8 RT owns blobs, RS indexes |
| Auth undefined | §9 key table |
| Mid-test cancel | §7 step table |
| Silent failure risk | D-020 `integration_event_log` on every edge |

---

## 11. Open questions

### Q-008-A: Should RT→RS test results write to legacy `timing_tests` or new `shared.timing_test_results`?

**Type:** schema
**Default:** New shared table; legacy RS job tests remain for historical jobs only.

### Q-008-B: Realtime vs poll for RS→RT reference sync?

**Type:** operational
**Default:** Poll delta every 15m + on-demand lookup at scan; Realtime optional later.

---

## 12. Comparison to Mike's workflow

| Workflow need | Contract hook |
|---------------|---------------|
| Testing-complete email | RW status `complete` + RT `overall_pass` on RS file (existing email flow via RW push) |
| Inspection photos on client file | RS inspection R2 + RT dial index side-by-side in rebuild inspection UI |
| Watchmaker analytics | RW pulls RT results by ref-serial + caliber (§3.2 RW pull — out of scope for RT push) |

---

## 13. Acceptance criteria check

- ✅ All four flows have proposed schemas and endpoints
- ✅ Photo storage decision: RT owns dial library
- ✅ Auth pattern proposed
- ✅ RT cancelled mid-test handling specified
- ✅ D-020 observability on every integration point
- ✅ D-019 gate noted for test-result attachment timing

---

_End of discovery. Q-008 complete._
