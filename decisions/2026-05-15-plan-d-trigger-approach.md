---
silo: 8
date: 2026-05-15
status: designed, not yet applied
last_validated: 2026-05-15
---

# Decision — Plan D in_progress trigger: DB trigger over frontend gating, skip backfill

## Question

How should the system bump parent `jobs.status` to `in_progress` and push to RS when any component starts work, so that band-only / refinish-only jobs stop appearing stuck pre-In Progress in RS's customer-facing timeline?

## Sub-decisions

### Approach: DB trigger over frontend gating

**Considered:**
- **Frontend-only gating** — modify `commitDrop` / `handleCommit` in ShopFloor.tsx + `BandKanban.handleMoveTo` to call `set_job_status` when transitioning a component to `in_progress`.
- **DB trigger** — `SECURITY DEFINER` trigger on `job_components` that fires on any UPDATE OF status to in_progress, from any writer.

**Chose:** DB trigger.

**Why:**
- Silo 8 audit found 14+ writers to `job_components`. Frontend-only would patch 2 (ShopFloor) or maybe 3 (+BandKanban) of them. The other 11 (StationScanner, BulkAssign, ComponentStatus, NewJob, BackfillBandJobs, component-creation.ts, rollisuite-intake, rollisuite-change-order, rollisuite-so-fulfilled) would silently miss the bump.
- Trigger catches every writer regardless of language (TS frontend, Deno edge function, SQL) and regardless of auth context (authenticated user vs service-role).

### Cannot reuse `set_job_status` RPC

**Why:**
- `set_job_status` has `auth.uid()` permission checks at function entry:
  ```sql
  IF auth.uid() IS NULL THEN RAISE EXCEPTION 'Not authenticated';
  IF NOT public.has_permission(auth.uid(), 'jobs.view') THEN RAISE EXCEPTION 'Insufficient permissions';
  ```
- Triggers fired by service-role writers (edge functions, intake) have `auth.uid() = NULL`. Calling `set_job_status` from a trigger would fail and abort the original component write, breaking the writer.
- Trigger must reimplement the write + RS push directly with `SECURITY DEFINER`.

### Auth pattern: hardcoded anon Bearer (match existing)

**Considered:** Hardcoded anon JWT (matches existing pattern), Supabase Vault secret, service role.

**Chose:** Hardcoded anon JWT.

**Why:**
- `set_job_status`, `assign_watchmaker`, `sync_watch_to_rollisuite` all use the same pattern. Consistency over architectural cleanliness for a v1 trigger.
- Vault would be cleaner but introduces a new pattern just for this trigger. Defer that change to a coordinated security pass across all RS-push functions.

### Bumpable-from states: include `uncased`

Currently uncased=0 rows in production, so harmless to include. `uncased` sits between `in_queue` and `in_progress` in `set_job_status`'s ordering, so semantically correct to bump from. `work_started_at` already guarded by `IS NULL` so won't overwrite legitimate work-start timestamps.

### Backfill: SKIP the 4 candidates

**Audit query result (silo 8):**
```sql
-- 4 jobs in waiting_approval with bracelet:in_progress:
-- EST-24800, 24185, 24392, 24187
-- All have work_started_at = NULL.
```

**Considered:**
- **Run backfill as written** — would push 4 jobs from `waiting_approval` to `in_progress` and fire 4 RS pushes.
- **Run backfill with trigger temporarily disabled** — bump jobs.status without RS push.
- **Skip backfill entirely.**

**Chose:** Skip.

**Why:**
- All 4 candidates are in `waiting_approval`. Bumping them to `in_progress` means telling the customer (via RS timeline) that work has started, before formal approval. Mike's call: that's not the message we want to send.
- The handoff doc Section 4 documents this workflow: "workers genuinely do start work before formal approval" is operational reality, but it's NOT something to surface in customer-facing RS. Internal vs external.
- The trigger going forward DOES include `waiting_approval` in the bumpable set — Mike OK'd future activity, just not the one-time backfill of these 4.

### Idempotency

- Early return when `new.status IS NOT DISTINCT FROM 'in_progress'` keeps trigger cheap on unrelated updates.
- Guard `tg_op = 'UPDATE' AND old.status = 'in_progress'` skips re-fires.
- Job-status guard means second component on the same job flipping to in_progress does nothing (no extra RS push).

## What would change these decisions

- If new writers appear later that need different bump behavior, revisit per-writer logic vs the universal trigger.
- If `waiting_approval` → `in_progress` becomes desired in customer-facing timeline (e.g. a workflow change where pre-approval work IS okay to advertise), the trigger doesn't need to change — only the operational policy on what gets surfaced.
- If RS Vault/secret pattern gets adopted broadly, refactor this trigger's hardcoded Bearer at the same time.
