# Q-002 — Map the Work Queue state machine (RW)

**Status:** in-progress
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-26
**Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**Work Queue State Machine** — The core job lifecycle engine in RolliWorking (RW). Every job in the shop passes through this state machine. It defines what statuses exist, how jobs transition between them, who/what can trigger each transition, and what side effects fire (emails, status pushes to RS and RC, component creation, etc.).

---

## 2. Enum of all statuses (canonical, from `set_job_status` RPC)

**Source:** `apps/rw/supabase/migrations/20260512161244_...sql` (latest migration defining the constraint)

```
intake → inspection → waiting_approval → in_queue → uncased → in_progress → parts_approval → parts_on_order → in_testing → finished
                    ↕                                        ↕
              (any time)                                (any time, bidirectional via downgrade)
              waiting_components
```

**All 11 statuses (in `set_job_status` DB constraint):**

| # | Status | Display Label | Timestamp column | Triggered by |
|---|---|---|---|---|
| 1 | `intake` | Intake | — | Auto by RS (incoming `rollisuite-intake`) |
| 2 | `inspection` | Inspection | — | Auto by RS (incoming `rollisuite-intake`) |
| 3 | `waiting_approval` | Awaiting Approval | — | Auto by RS (new job = `waiting_approval`) |
| 4 | `in_queue` | In Queue | — | Manager sets from Work Queue |
| 5 | `uncased` | Uncased | `uncased_at` | Watchmaker sets via RW dropdown |
| 6 | `in_progress` | In Progress | `work_started_at` | Watchmaker sets (first hands-on work) |
| 7 | `parts_approval` | Parts Pending Approval | — | Watchmaker sets (requesting parts) |
| 8 | `parts_on_order` | Parts on Order | — | Admin/Mike sets after ordering |
| 9 | `in_testing` | In Testing | `in_testing_at` | Watchmaker sets (testing phase) |
| 10 | `waiting_components` | Waiting on Components | — | Watchmaker sets (missing components) |
| 11 | `finished` | Complete | `finished_date` | Manager/WM sets (work complete) |

---

## 3. Transitions (complete map)

### T0: RS → RW (incoming intake) — auto-transition on job creation

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| *(none)* | `waiting_approval` | RS `rollisuite-intake` webhook fires | RS creates job via `rollisuite-intake` edge function | Creates `jobs`, `watches`, `customers` rows; auto-creates `job_components` based on service code flow (STANDARD_FLOW: watch_head + case; SPLIT_FLOW: + bracelet; BAND_ONLY_FLOW: bracelet only); sends "estimate sent" email to client |

**Source:** `apps/rs/supabase/functions/rollisuite-intake/index.ts` (RS → RW via `rollisuite-intake` edge function in RW)

### T1: `in_queue` → `uncased`

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| `in_queue` | `uncased` | Manager assigns to watchmaker; watchmaker uncases | Manager via Work Queue `BulkAssign`; watchmaker via `set_job_status` RPC | Sets `uncased_at = now()`; pushes status to RS via `rollisuite-status-push` |

### T2: `uncased` → `in_progress`

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| `uncased` | `in_progress` | Watchmaker starts work | Watchmaker via RW dropdown (iPad) | Sets `work_started = true`, `work_started_at = now()`; **triggers email** (noted via `in_progress` email needed path in `WorkQueue.tsx`) |

### T3: `in_progress` → `in_testing`

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| `in_progress` | `in_testing` | Watchmaker starts testing | Watchmaker via RW dropdown (iPad) | Sets `in_testing_at = now()`; triggers "in_testing" email to client |

### T4: `in_testing` → in_progress / in_queue / parts_approval / parts_on_order (downgrade)

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| `in_testing` | `in_progress` | Watchmaker downgrades (found issue) | Watchmaker via RW dropdown | Sets `status_updated_at`; triggers "downgraded" email if no email sent since `in_testing_at` |
| `in_testing` | `in_queue` | Downgrade | Watchmaker via RW dropdown | Same — downgrade detected when `in_testing_at` exists but status is earlier, per `WorkQueue.tsx` line ~161 |
| `in_testing` | `parts_approval` | Downgrade | Watchmaker | Same pattern |
| `in_testing` | `parts_on_order` | Downgrade | Watchmaker | Same pattern |

**Detection logic:** `WorkQueue.tsx` line ~161: if `job.in_testing_at` exists but current status is `in_progress`, `in_queue`, `parts_approval`, or `parts_on_order`, it's flagged as downgrade with email notification.

### T5: `in_progress` or any status → `parts_approval`

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| *(any)* | `parts_approval` | Watchmaker submits parts request | Watchmaker via `PartsRequest` page | Creates a parts request record on the RW side; calls `submit-parts-approval` edge function; pushes to RS via `rollisuite-parts-approved` edge function |

### T6: `parts_approval` → `parts_on_order`

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| `parts_approval` | `parts_on_order` | Mike enters price + sends approval to client | Admin (Mike) in `ApprovePartsRequest` | Client approves → order placed; triggers `rw-parts-approved` event to RS |

### T7: `waiting_components` → in_progress / in_queue

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| `waiting_components` | `in_progress` | Components received | Watchmaker sets status via dropdown | Work resumes |
| `waiting_components` | `in_queue` | Components received | Manager via Work Queue | Same, but manager-initiated |

### T8: Any status → `finished`

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| *(any)* | `finished` | Job complete | Manager via Work Queue / Dashboard | Sets `finished_date = CURRENT_DATE`; **requires**: all non-cancelled components must have `status IN ('waiting_components', 'reunited')` (validation in `set_job_status` RPC); pushes to RS via `rollisuite-status-push`; pushes to RS via `rollisuite-job-finished` via caret-delimited payload; triggers "testing complete" / final email |

### T9: `waiting_approval` → `in_queue` (client-approved estimate)

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| `waiting_approval` | `in_queue` | Client approves estimate (from RC portal or RSVP) | Client via RC portal; auto when RS receives approval | Status moves to `in_queue`, then manager assigns |

### T10: `in_queue` → `waiting_components` (rare path)

| From | To | Trigger | Who/What | Side effects |
|---|---|---|---|---|
| `in_queue` | `waiting_components` | Watchmaker notices missing component mid-job | Watchmaker sets status via dropdown | Same as T7 pattern |

---

## 4. Who can trigger each transition

| Transition | Who can do it |
|---|---|
| T0 (intake → waiting_approval) | **Automatic** (RS webhook `rollisuite-intake`) |
| T1-T2 (in_queue → uncased → in_progress) | **Watchmaker** (RW app, iPad dropdown) |
| T3 (in_progress → in_testing) | **Watchmaker** (RW app, iPad dropdown) |
| T4 (in_testing → downgrade) | **Watchmaker** (RW app, iPad dropdown) — intentional, Mike named this |
| T5→T6 (parts_approval → parts_on_order) | **Watchmaker** submits → **Mike/Admin** approves |
| T7 (waiting_components → back) | **Watchmaker** or **Manager** |
| T8 (→ finished) | **Manager** (Work Queue / Dashboard) or watchmaker who auto-completes |
| T9 (waiting_approval → in_queue) | **Automatic** (client approval) or **Manager** |

### Auth model for `set_job_status`

```sql
-- set_job_status(job_id uuid, new_status text)
-- SECURITY DEFINER function
IF auth.uid() IS NULL → RAISE EXCEPTION
IF NOT public.has_permission(auth.uid(), 'jobs.view') → RAISE EXCEPTION
-- (The actual permission check uses has_permission RPC)
```

---

## 5. Cross-app side effects per transition

### Push to RS (RolliSuite) — `rollisuite-status-push`

| When | Edge function | Payload | Destination |
|---|---|---|---|
| **Every time** `set_job_status` fires in RW | `apps/rw/supabase/functions/rollisuite-status-push/index.ts` | `{ estimate_number, status, updated_at, work_started_at, in_testing_at, finished_date }` | RS `job-status-update` edge function |

**Status mapping (RW → RS):**

| RW status | RS local status |
|---|---|
| `in_queue` | `on_hand` |
| `uncased` | `on_hand` |
| `in_progress` | `on_hand` |
| `in_testing` | `on_hand` |
| `parts_approval` | `on_hand` |
| `parts_on_order` | `on_hand` |
| `waiting_approval` | `on_hand` |
| `waiting_components` | `on_hand` |
| `finished` | `ready_to_ship` |

**Important:** RS maps ALL RW internal statuses to **one** of two RS statuses: either `on_hand` or `ready_to_ship`. This is a heavy lossy mapping — RS loses the distinction between `in_progress`, `in_testing`, `parts_approval`, etc.

### Pull from RS to check RW status — `rw-job-status`

| When | Edge function | What it does |
|---|---|---|
| Daily cron at 5am (`rw-status-cron`) | `apps/rs/supabase/functions/rw-status-cron/index.ts` | Batch-fetches all pending inspection statuses from RW; caches in `cache_store` |

**RW response payload (`RW status /job-status` function):**

| Field | Values |
|---|---|
| `status` (external) | `in_queue`, `in_progress`, `in_testing`, `complete` |
| `status_raw` (internal) | Full canonical status: `in_queue`, `uncased`, `in_progress`, `in_testing`, `parts_approval`, `parts_on_order`, `waiting_components`, `finished` |
| `status_label` (display) | `In Queue`, `Uncased`, `In Progress`, `In Testing`, `Parts Pending Approval`, `Parts on Order`, `Waiting on Components`, `Complete` |
| `inspection_completed_at` | ISO timestamp |
| `status_updated_at` | ISO timestamp |
| `finished_date` | ISO timestamp |
| `emails_sent` | Array of `{ type, sent_at, subject }` |
| `parts_approved_at` | ISO timestamp |
| `uncased_at` | ISO timestamp |
| `status_history` | Array of `{ status, changed_at }` |

### Push to RC (RC timeline status)

| When | Edge function | What it does |
|---|---|---|
| RC needs client timeline | `apps/rs/supabase/functions/rc-timeline-status/index.ts` (RS-side) | Queries job activity log, fetches RW status; builds timeline sections per flow type (STANDARD_FLOW, SPLIT_FLOW, BAND_ONLY_FLOW); caches 60s in `cache_store` |

**RC display status labels (from `rc-timeline-status`, RS-side):**

| RW status | RC display step ID | RC display label |
|---|---|---|
| `in_queue` | `rw_in_queue` | In Queue |
| `uncased` | `rw_uncased` | Uncased |
| `in_progress` | `rw_in_progress` | In Progress |
| `in_testing` | `rw_in_testing` | In Testing |
| `parts_approval` | `rw_parts_pending` | Parts Pending Approval |
| `parts_on_order` | `rw_parts_on_order` | Parts on Order |
| `finished` | `rw_complete` | Work Complete |

### Parts request → approval push

| When | Edge function | What it does |
|---|---|---|
| Mike approves parts | `apps/rw/supabase/functions/rollisuite-parts-approved/index.ts` | Creates `PARTS_APPROVED` event payload; pushes to `apps/rs/supabase/functions/rw-parts-approved/index.ts` |

**RS-side `rw-parts-approved` actions:**
1. Rate-limits via `check_rate_limit` RPC
2. Idempotency check on `received_webhook_events.event_id`
3. Finds estimate in `estimates` table (multi-format matching)
4. Appends note to `estimates.internal_notes`: `"✓ Parts approved on RW: {partsList} - {formattedDate} (by {approvedBy})"`
5. Marks event as `processed` in `received_webhook_events`
6. **Does NOT** write to `customer_memo` or SO (internal only, per code comment)

### Watchmaker assignment push

| When | Edge function | What it does |
|---|---|---|
| Manager assigns in Work Queue | `apps/rw/supabase/functions/rollisuite-watchmaker-assign/index.ts` | Pushes to `apps/rs/supabase/functions/rw-watchmaker-assignment/index.ts` |

**RS-side `rw-watchmaker-assignment` actions:**
1. Appends `🔧 Watchmaker assigned: {name} — {date} (status: {status})` to `estimates.internal_notes`
2. Writes to `audit_log` table
3. Duplicates detection: skips if same watchmaker already noted

### Work completed push (fire-and-forget)

| When | Edge function | What it does |
|---|---|---|
| Job set to `finished` | `apps/rw/supabase/functions/rollisuite-job-finished/index.ts` | Calls RS via `rollisuite-job-finished` edge function; **always returns 200** to client regardless of external result |

**Payload format:** Caret-delimited `estimateNumber^referenceNumber^finishedDate` (legacy format) or JSON body.

### Inbound webhook processing (from external → RW)

| When | Edge function | What it does |
|---|---|---|
| External RolliSuite event | `apps/rw/supabase/functions/rollisuite-webhook/index.ts` | Handles `WORK_COMPLETED` (sets status to `finished`) and `STATUS_UPDATE` mapping from RS status codes (`PENDING→in_queue`, `IN_PROGRESS→in_progress`, `TESTING→in_testing`, `COMPLETED→finished`, `WAITING_PARTS→parts_on_order`) |

### RS `job-status-update` edge function (inbound from RS)

| When | Edge function | What it does |
|---|---|---|
| RS pushes back | `apps/rs/supabase/functions/job-status-update/index.ts` | Maps RS `on_hand`/`ready_to_ship` to RW local status; writes to `jobs` table |

**Note:** This function is in the RS app directory but IS an RW-target function. RS maps back the local statuses: `on_hand` and `ready_to_ship` — but the reverse mapping back to RW statuses was not found in this code path. It seems RS is primarily a **status consumer** of RW, not a status writer.

---

## 6. Parts request flow (start to finish)

```
┌─────────────────────┐
│  Watchmaker (iPad)  │
│  PartsRequest page  │
│                     │
│  1. Types parts desc│
│  2. Submits request │
│  3. Request logged  │  ← stored in RW
└────────┬────────────┘
         │ rollisuite-parts-approved
         │ EVENT: PARTS_APPROVED
         ▼
┌─────────────────────────┐
│  Mike / Admin (RW)      │
│  Approves parts in UI   │
│  Sets price             │
│  Sends approval to      │
│  client via email       │
└────────┬────────────────┘
         │ RC sends approval
         │ (client accepts)
         ▼
┌─────────────────────────────┐
│  RS rw-parts-approved       │
│  edge function              │
│                             │
│  - Rate limits              │
│  - Idempotency check        │
│  - Updates est.internal_n   │
│    otes                     │
│  - Idempotent (event_id)    │
└─────────────────────────────┘
```

**RW → RS email push:** On parts approval, RS `rw-parts-approved` function writes to `estimates.internal_notes` as a text note — **no auto-email** is sent to the client from RS (per comment "Parts approval notes are internal-only").

---

## 7. Email side effects

### Automated emails triggered by status transitions

| Status transition | Email triggered by | What |
|---|---|---|
| → `in_progress` | `WorkQueue.tsx` `getStatusEmailNeeded` (line ~155+) "email needed" logic | Client notified work started (via `send-inspection-email` or equivalent) |
| → `in_testing` | Same "email needed" logic | Client notified testing started |
| → downgrade from `in_testing` | Same logic (downgrade detected when `in_testing_at` exists but status is earlier) | Client notified status changed |
| → `waiting_approval` (intake) | `rollisuite-intake` (RS-side) | "Awaiting your reply before adding job to our work queue" — estimate sent to client |
| → `finished` | `send-inspection-email` / manual "testing complete" email | Final testing complete notification to client |
| Parts approved | None (internal note only in RS) | — |

**Email flow:** Mike sends "testing complete" email from RW. Wanted: bulk email page (scan all finished jobs, one button → bulk email to clients) per Michael's workflow doc.

---

## 8. Job component creation (from intake)

When a job is first created via `rollisuite-intake` (RS → RW), the RW side creates `job_components` records:

| Flow | Components created |
|---|---|
| `STANDARD_FLOW` | `watch_head` (label: `{ref}`), `case` (label: `{ref}-C`) |
| `SPLIT_FLOW` | `watch_head` (label: `{ref}`), `case` (label: `{ref}-C`), `bracelet` (label: `{ref}-B`) |
| `BAND_ONLY_FLOW` | `bracelet` (label: `{ref}-B`) |

All components start with `status = 'in_queue'`.

**Validation for `→ finished` transition:** `set_job_status` RPC requires all non-cancelled components to have `status IN ('waiting_components', 'reunited')`. This is the **only** guard against finishing before components are accounted for.

---

## 9. Gaps found

### Gap 1: Heavy lossy status mapping (RW → RS)

**Finding:** RS maps 9 RW statuses to just 2 statuses: `on_hand` or `ready_to_ship`. The RS Daily Hit List page (which reads RW status) loses all granular distinction — it cannot tell if a job is in progress, in testing, or waiting for parts; only "on hand" or "ready to ship."

**Evidence:** `apps/rs/supabase/functions/rw-status-cron/index.ts` and `apps/rs/supabase/functions/job-status-update/index.ts` both use the same 2-value mapping.

### Gap 2: No auto-transition from `waiting_approval` to `in_queue`

**Finding:** `waitng_approval` jobs created by RS intake have no automatic transition path to `in_queue`. They remain `waiting_approval` until a manager manually approves the estimate and sets the status. There is **no "client approved" event** that auto-triggers this transition.

**Evidence:** Michael's workflow doc says "Job status flow (Q-002 output) for 'in safe' / 'in workshop' / 'in polish' states" — but this gap means jobs stay in `waiting_approval` indefinitely after client approval unless a manual step occurs.

### Gap 3: `uncased` is a dead-end from RS perspective

**Finding:** `uncased` sets a timestamp (`uncased_at`) but does not change the semantic meaning for any RS downstream consumer. RS maps it to `on_hand`, same as `in_queue`, `in_progress`, `in_testing`, `parts_approval`, and `parts_on_order`. The `uncased` status is RW-only metadata.

### Gap 4: `waiting_components` has no auto-recovery

**Finding:** Jobs that go to `waiting_components` stay there indefinitely until someone manually moves them. There is no mechanism to re-notify the client or auto-push to a queue once components arrive from the client.

### Gap 5: No audit log for in-app RW status changes

**Finding:** `set_job_status` is a pure DB function — it does NOT produce a `job_activity_log` entry (unlike the RS-side `job_status-update` which writes to `job_activity_log`). RW status changes are invisible to the `job_activity_log` unless a status update happens through the RS-side `job-status-update` path (which is inbound from RW, not outbound).

### Gap 6: `rollisuite-webhook` inbound mapping uses different RS status codes

**Finding:** The `rollisuite-webhook` function on RW receives `STATUS_UPDATE` events from RS with status values like `PENDING`, `IN_PROGRESS`, `TESTING`, `COMPLETED`, `WAITING_PARTS` — a completely different naming convention than the RW internal enum. This creates a second parallel status vocabulary.

### Gap 7: Missing "ready-for-inspection" and "ready-to-ship" from RW enum

**Finding:** Mike's workflow doc named "ready-for-inspection" and "ready-to-ship" as expected statuses. These do **not** exist in the RW enum. "ready-to-ship" only maps from `finished` in RS, not from any RW status directly.

---

## 10. Edge cases & failure modes

### `set_job_status` fires during a `finished` transition but components aren't all reunited

**Handling:** `RAISE EXCEPTION 'Cannot finish: not all components are reunited'`. The DB blocks the transition, but the UI layer in RW must surface this error to the watcher.

### `rollisuite-status-push` fails externally

**Handling:** Always returns `success = true` to the RW caller — non-blocking. The external sync failure is silently swallowed. RS is expected to catch up via the 5am daily cron (`rw-status-cron`).

### Estimate not found in `rw-parts-approved` handler

**Handling:** Returns `received: true, queued: true, reason: 'estimate_not_found'` — silently discards the data with no retry mechanism.

### No email sent when client needs one

**Finding:** Email sending is triggered by **frontend logic** in `WorkQueue.tsx` (the "email needed" notification paths). If the frontend doesn't call the send function, no email fires. There is a gap: if a manager manually sets status changes directly in the DB without going through the RW UI, no emails are sent.

---

## Schema summary — `jobs` table columns relevant to state machine

| Column | Type | Purpose |
|---|---|---|
| `id` | UUID | PK |
| `status` | TEXT | Current status (constrained to 11 enum values) |
| `estimate_number` | TEXT | Keying reference |
| `work_started` | BOOLEAN | Shortcut flag for "work has started" |
| `work_started_at` | TIMESTAMPTZ | When `in_progress` started |
| `in_testing_at` | TIMESTAMPTZ | When `in_testing` started |
| `uncased_at` | TIMESTAMPTZ | When `uncased` happened |
| `finished_date` | DATE | When `finished` was set |
| `updated_at` | TIMESTAMPTZ | Last status change timestamp |
| `sent_email_templates` | JSONB array | Email send history |
| `last_update_email_sent` | TIMESTAMPTZ | Last email sent timestamp |

**`job_components` table (validation dependency for `finished`):**

| Column | Type | Purpose |
|---|---|---|
| `job_id` | UUID FK → `jobs` | Parent job |
| `component_type` | TEXT | `watch_head`, `case`, `bracelet` |
| `status` | TEXT | `'in_queue'`, `'waiting_components'`, `'reunited'`, `'cancelled'` |
| `label_id` | TEXT | Barcode label reference |
| `department` | TEXT | Where component is physically |

---

## Comparison to Mike's workflow doc

### What Mike named that exists in code

Mike's named statuses:
- **In testing** ✅ Exists as `in_testing`
- **Waiting parts approval** ✅ Exists as `parts_approval` (mapped to "Parts Pending Approval")
- **Downgrade from in-testing back to in-progress** ✅ Supported (T4 transition)
- **In queue** ✅ Exists as `in_queue`
- **In progress** ✅ Exists as `in_progress`

### What Mike named that's missing from code

- **"Ready for inspection"** ❌ Not in enum — there's no equivalent status. Mike expected a status marking the job ready for the client to inspect (e.g., via photos or preview), but the RW enum jumps straight from `in_testing` → `finished`.
- **"Ready to ship"** ❌ Only exists in RS as `ready_to_ship` (mapped from RW's `finished`), not as a distinct RW status.
- **"Downgrade"** ❌ Not a status — it's a transition pattern detected on the RW frontend (`WorkQueue.tsx`), not a database concept.
- **"In safe" / "in workshop" / "in polish"** ❌ Not in enum. Physical location tracking is not in the state machine.

### Watchmaker workflow gaps per Michael's doc

1. **No "scan ref-serial then scan process QR" pattern exists** — Michael named this as needed. Currently, watchmakers use a dropdown menu on the iPad.
2. **No voice-to-text for job notes** — Michael named this as needed.
3. **Manager sets status, not watchmaker** — Michael's workflow doc says managers manage the Work Queue, but watchmakers also trigger status changes via dropdown. The boundary between manager and watchmaker permissions is via `has_permission(auth.uid(), 'jobs.view')` which is insufficient — there's no explicit check for write permissions on `set_job_status`.
