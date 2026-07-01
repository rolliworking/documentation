# DRAFT — Canonical Data Model (SPEC-002 starting point)

> **⚠️ DRAFT — DO NOT BUILD YET**
>
> This document is a starting point for SPEC-002 weekend planning. It is **not locked**, **not reviewed**, and **not approved** for migration or Emergent execution. Expect significant revision after SPEC-PREP human answers (hosting, migration cutoff, scope hold).
>
> **Purpose:** Save 2–3 hours by providing table shapes, relationships, and decision traceability instead of a blank page.
>
> **Sources:** D-010, D-011, D-019, D-020, D-021, D-022, discovery Q-001 through Q-015, A-20260629-* answers.

---

## Design principles

1. **One identity per watch** — serial number is the spine (D-022)
2. **Pieces express assembly state** — X of Y, not competing client_property vs job_components rows
3. **Stage 1 vs Stage 2 intake** — possession separate from inventory commit (D-019)
4. **Photos are indexed, not siloed** — `shared.intake_photos` bridges storage backends (D-021)
5. **Every write is auditable** — `shared.audit_log` for D-020 / D-015 compliance
6. **Living masters with promotion** — `status` column on reference tables (D-011)

---

## Entity relationship (core)

```
shared.customers 1───* shared.watches 1───* shared.pieces
        │                    │                    │
        │                    │                    └──► shared.stations (station_id)
        │                    │
        │                    *─── shared.intake_photos
        │
        *─── rc.conversations (rc schema — not detailed here)

rs.intake_possession *───1 shared.watches (after Stage 2 commit)
rs.intake_possession ───► rs.estimates (optional FK — Q-015 no-estimate path)
```

---

## Table: `shared.customers`

Canonical customer master. Supersedes fragmented RC/RS client rows over time (Q-012).

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| display_name | text NOT NULL | |
| primary_email | text | Case-insensitive unique index (Q-012) |
| primary_phone | text | |
| qbo_customer_id | text | Nullable — QBO link |
| household_id | uuid FK | Nullable — W-32 spouse/household |
| status | text | `active`, `merged`, `archived` |
| promotion_status | text | D-011: `stub` → `authoritative` |
| created_at | timestamptz | |
| updated_at | timestamptz | |
| validated_by | uuid FK → shared.staff | D-011 |
| validated_at | timestamptz | |

**Owner:** Vianna (D-011). **Merge policy:** TBD in SPEC-002 from Q-012 answer.

---

## Table: `shared.watches`

One row per unique serial number — the identity spine (D-022, A-20260629-010).

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| customer_id | uuid FK → shared.customers | |
| brand | text | |
| model | text | |
| reference_number | text | Links to shared.reference_models |
| serial_number | text NOT NULL | **Unique** — identity key |
| custody_status | text | `in_service`, `in_safe`, `with_customer`, `awaiting_parts`, etc. |
| current_estimate_id | uuid | Nullable — RS estimate FK |
| notes | text | |
| created_at | timestamptz | |
| updated_at | timestamptz | |

**Replaces over time:** RS `client_property` as authoritative identity (migration mapping TBD).

---

## Table: `shared.pieces`

Current-state assembly pieces for a watch (D-022).

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| watch_id | uuid FK → shared.watches NOT NULL | |
| piece_number | int NOT NULL | 1-based |
| piece_total | int NOT NULL | Y in "X of Y" |
| piece_type | text NOT NULL | `whole_watch`, `head`, `bracelet`, `case`, `dial`, `bezel`, `movement`, … |
| station_id | text FK → shared.stations | Includes `safe_bin_A1` etc. (A-20260629-009) |
| staff_id | uuid FK → shared.staff | Current custodian |
| moved_at | timestamptz | Last movement |
| status | text | `active`, `closed` (part swap — old piece closed) |
| created_at | timestamptz | Set at Receive Watch Stage 2 (A-20260629-007) |

**Constraints:**
- When whole: exactly one row with piece_number=1, piece_total=1, piece_type=`whole_watch`
- When disassembled: sum of active rows = piece_total
- Reunification: collapse to single `whole_watch` row (transaction + audit)

**Replaces over time:** RW `job_components` as location authority.

---

## Table: `shared.stations`

Physical and logical locations — unified namespace (A-20260629-009).

| Column | Type | Notes |
|--------|------|-------|
| station_id | text PK | e.g. `wm_bench_1`, `safe_bin_A1`, `receiving`, `pickup` |
| display_name | text | |
| department | text | `watchmaker_bench`, `safe_storage`, `polish_room`, `receiving`, `testing`, … |
| sort_key | text | Vianna due-date sort (UI concern) |
| is_active | boolean | |
| facility_zone | text | Nullable — links to floor plan (SPEC-PREP-007) |

---

## Table: `shared.staff`

Staff directory for custody, audit attribution, role-based auth (D-023).

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | Links to auth.users |
| display_name | text | |
| role | text | `owner`, `manager`, `watchmaker`, `front_desk`, `kiosk`, … |
| is_active | boolean | |
| created_at | timestamptz | |

**Note:** Permissions are role-based, never hardcoded user IDs (D-023, SPEC-PREP-004).

---

## Table: `shared.intake_photos`

Canonical photo index (D-021). Apps query here first; storage backend is opaque.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| watch_id | uuid FK → shared.watches | |
| job_id | uuid | Nullable — RS job/estimate context |
| photo_type | text | `dial`, `caseback`, `bracelet`, `whole_watch`, … |
| storage_location | text | `R2`, `supabase`, `dial_library` |
| storage_path | text | Backend-specific key |
| captured_at | timestamptz | |
| captured_by | uuid FK → shared.staff | |
| stage | text | `intake`, `inspection`, `pickup`, `testing` |
| created_at | timestamptz | |

**Rule:** No index row = photo invisible to apps. Every new capture writes index + file (D-021).

---

## Table: `shared.audit_log`

Cross-app audit surface (D-019, D-020, D-015).

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| event_type | text | `piece_moved`, `intake_variance`, `pickup_verified`, `qbo_sync`, … |
| watch_id | uuid FK | Nullable |
| piece_id | uuid FK | Nullable |
| actor_id | uuid FK → shared.staff | |
| from_station | text | Nullable |
| to_station | text | Nullable |
| payload | jsonb | Event-specific detail |
| created_at | timestamptz | |

---

## Table: `shared.timing_test_results` (proposed — A-20260629-003)

New shared table for RolliTime results. Legacy RS `timing_tests` / `pressure_tests` stay read-only.

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| watch_id | uuid FK | |
| test_type | text | `timing`, `pressure`, `amplitude`, … |
| before_values | jsonb | |
| after_values | jsonb | |
| passed | boolean | |
| tested_at | timestamptz | |
| tested_by | uuid FK → shared.staff | |
| source | text | `rollitime`, `manual_import` |
| rt_session_id | text | External RT reference |

---

## Table: `rs.intake_possession` (Stage 1 — D-019)

| Column | Type | Notes |
|--------|------|-------|
| id | uuid PK | |
| source | text | `shipping_receive`, `drop_off` |
| customer_id | uuid FK | |
| estimate_id | uuid FK | Nullable — no-estimate path (Q-015) |
| tracking_number | text | Shipping only |
| item_count_claimed | int | |
| received_by | uuid FK → shared.staff | |
| received_at | timestamptz | |
| status | text | `received`, `needs_review`, `variance` |
| escalated_at | timestamptz | Q-015-A stagnation tracker |
| notes | text | |

**Stage 2 link:** `rs.receive_watch_sessions` references possession_id; on commit creates `shared.pieces`.

---

## Migration notes (draft — not executable)

| Legacy | Target | Strategy |
|--------|--------|----------|
| RS `customers` | `shared.customers` | Merge with RC clients; dedupe email |
| RS `client_property` | `shared.watches` + `shared.pieces` | Map serial → watch; initial 1-of-1 piece |
| RW `job_components` | `shared.pieces` | Map station/status → station_id |
| Supabase Storage URLs | `shared.intake_photos` | Backfill script; no file move (D-021) |
| R2 inspection photos | `shared.intake_photos` | Backfill script |

**Cutoff date:** TBD — SPEC-PREP-002.

---

## Open questions blocking SPEC-002 lock

| ID | Topic |
|----|-------|
| SPEC-PREP-001 | Same Supabase project vs new |
| SPEC-PREP-002 | Historical migration depth |
| Q-012-A | Conversation reuse (customer linking) |
| Q-015-A | No-estimate escalation days |

---

## Transferability Test preview (not final)

| Test | Draft result | Watch item |
|------|--------------|------------|
| Absence | Pass if roles not names | Approval thresholds in SPEC |
| Mike | N/A at schema layer | Reference data promotion via RolliCurator |
| Onboarding | Pass if runbooks exist | Per-workflow docs required |
| Handoff | Pass | Auditable pieces + audit_log |
| Multiplication | Pass | station_id data-driven per location |

---

_End of draft. SPEC-002 authors: revise, run Transferability Test, then lock._
