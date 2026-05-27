---
silo: 8
date_range: 2026-05-14 to 2026-05-15
status: partial — multiple shipped, multiple designed-but-not-applied
last_validated: 2026-05-15
---

# Silo 8 — Multi-thread session: Pills, SO unblock, Shove design, Plan D design

Long session spanning late evening May 14 into afternoon May 15. Two real-time ships (Job History Lookup component pills on RW; SO 500950 Ship Station unblock + Sync QBO button widening on RS). Several designs scoped but deferred to apply (Plan D in_progress trigger; Plan E Shove to Ship Station; Fix B re-entry guard still not landed after silo 7+8). Surfaced major architectural finding (only 2 of 14+ writers populate station_id) and two new urgent operational items (QBO Customer Memo length error on SO 501718; QBO webhook CloudEvents deadline today).

---

## 1. What shipped

### Verified live on RW production

**Job History Lookup component status pills.** `src/components/jobs/JobHistoryLookup.tsx` only. Per-component pills below existing 3 header pills (status / due / inspected). Reads `job_components` with cancelled rows filtered, fetches once per dialog selection. Reuses existing `Badge` component from `@/components/ui/badge` with `variant="secondary"`. Fallback precedence: `assignee_initials` → titlecased `department` → status alone. Sort order: watch_head → case → bracelet → alphabetical. Multi-component jobs render one pill per component. Empty/orphan jobs show nothing extra.

Verified apply via Lovable confirmation: state at line 277 (new state), resets at 290/379/508/517, components fetch added in `selectJob`, header JSX extended. No new imports (Badge already imported). Smoke test not explicitly reported back; recommended on EST-24392 (single-bracelet) and any multi-component job.

### Verified live on RS production

**SO 500950 unblock Part 1 (data UPDATE).**
```sql
UPDATE public.sales_orders
SET is_paid = true, paid_at = now()
WHERE id = 'a5e827c2-1eb9-4094-b3ba-c79d020d3667';
```
QBO had already confirmed `balance=0, isPaid=true` via live `qbo-invoice-status` call. Row now passes the queue's isPaid gate and appears Ready.

**Sync QBO button window widening Part 2.** `src/components/sales/PaidUnshippedOrdersList.tsx` lines 350-358 + 414. Replaced 7-day-recent filter with "any unpaid row with `qboInvoiceId` in visible queue, oldest-first, cap 25." Toast reference updated `recentRows.length` → `unpaidRows.length`. Lovable confirmed apply — no other files touched.

### Carried forward from silo 7 (test branch, still pending merge)

- **Performance Report `client_first_name`** — status unchanged from silo 7. Smoke test still pending.
- **Fix A persistOrder UUID patch-back** — silo 7 test 1 passed; test 2 (with navigation) status uncertain.

---

## 2. What was learned (the texture)

### Architectural findings

**14+ writers to `job_components` exist; only 2 set `station_id`.** Silo 8 audit found that ShopFloor.tsx's `commitDrop` (line 664) and `handleCommit` (line 1033) are the only writers populating `station_id`. Twelve other writers do not:

| File | Op | Writes station_id? |
|---|---|---|
| `ShopFloor.tsx:1576` smart-create | INSERT | No |
| `NewJob.tsx:694` auto-create bracelet | INSERT | No |
| `BandKanban.tsx:242, 503, 520, 556, 968` | UPDATE×4 + INSERT | No |
| `ComponentStatus.tsx:557, 607, ~810` | INSERT×3 | No |
| `BackfillBandJobs.tsx:110, 179` | INSERT + UPDATE | No |
| `StationScanner.tsx:176, 302` | UPDATE×2 | No |
| `component-creation.ts:258, 268, 279` | UPSERT/UPDATE | No |
| `rollisuite-intake/index.ts:577, 585, 594` | INSERT×3 | No |
| `rollisuite-change-order/index.ts:99, 141, 154` | INSERT + UPDATE×2 | No |
| `rollisuite-so-fulfilled/index.ts:75` | UPDATE | No |

`rollisuite-intake` is the primary source of watch_head/case/bracelet rows. BandKanban:968 produces recent NULL bracelets. Result: 221/222 components have NULL station_id; 130/136 logs have NULL station_id (of which 42 are `backfill` rows where NULL is correct by design).

**Implication:** any surface reading station_id must treat NULL as the dominant case, not the exception. Department fallback is the primary display path; station_id is opportunistic. Pills shipped silo 8 deliberately do not read station_id for this reason.

**`handleCommit` is verified fully patched.** Silo 7's claim was technically correct — both commitDrop (line 672) and handleCommit (line 1045) write `station_id` to their job_component_logs inserts. The absence of scan-path station_id rows in production data is "scan flow not yet exercised since the patch," not "patch missing."

**RW and RS have different role schemas.**
- RW: `owner`, `manager`, `staff`. No `admin` role exists.
- RS: `admin`, `manager`, `office`, `front_desk`, `staff`, `band_room` (AppRole enum at `src/types/database.ts:13`). No `owner` role exists.
- In RS context, `admin` IS the super-user / owner-equivalent. Gate super-user features on `isAdmin` from `useAuth()`.
- In RW context, owner is super-user; manager has 25/28 owner permissions.

**Ship Station is NOT a real external integration.** "Ship Station" refers to the internal `/sales/ship-station` queue page (`PaidUnshippedOrdersList.tsx`). "Push to Ship Station" on the SO detail page (lines 2069-2093) is a local DB UPDATE only:
- Always sets `fulfillment_channel = 'ship'`
- On downgrade from `status='shipped'`: clears `tracking_number`, `ship_date`, zeroes all `so_lines.shipped_qty` — **destructive side effect, no confirmation dialog**
- Does NOT read `is_paid`, `paid_at`, `balance_due`, or any QBO field
- Only gate is `disabled={isNew}` (SO must be saved)
- The queue's isPaid gate (`o.is_paid === true || (!!o.paid_at && Number(o.balance_due || 0) === 0)`) is independent — push does NOT bypass it.

**Single-dimensional `jobs.status` is a real visibility limitation for RS.** Band-only / refinish-only jobs can have a component actively `in_progress` while parent `jobs.status` stays `in_queue` or `waiting_approval`, leaving RS's customer-facing timeline stuck pre-In Progress. Plan D's trigger fixes the "→ in_progress" bump. It does NOT fix "one component done, parent partially advanced" — there's no obvious target status, would require an architectural change.

**`payment_bypass_log` is the right audit surface for Shove.** Already has `sales_order_id, so_number, customer_id, bypass_amount, balance_due, station, bypassed_by, bypassed_by_email, notes, created_at`. RLS: any auth can insert, only admins can read. SMS alert already wired via `send-security-alert`. Reuse with `station='ship_station'` rather than create new `sales_order_overrides` table.

**`set_job_status` RPC cannot be reused from triggers.** Has `auth.uid()` permission checks at function entry (`IF auth.uid() IS NULL THEN RAISE EXCEPTION 'Not authenticated'`). Service-role writers (edge functions, intake) would fail. Plan D's trigger reimplements the write + RS push directly with SECURITY DEFINER.

**QBO sync has a small window that lets paid invoices fall through.** Cron checks 21 oldest open invoices per run; pre-silo-8 button only re-checked last 7 days. SO 500950 (44 days old, paid in QBO, locally unpaid) sat in the gap. Part 2 fixes the button side. Cron is still 21-row FIFO — frozen file.

### Operational reality (surfaced via package-flow research at end of session)

- `BulkArrivalScanPage.tsx` is dead code. Consolidated into `ReceivePackagesPage.tsx` per `App.tsx:74`. Still in repo, still imports same hooks; re-enabling its route would create a parallel UI without photo/component/email steps.
- Two parallel tables track packages: `package_arrival_scans` (counter with `memo` as free-text mirror) and `package_scan_logs` (rich record with `received_items text[]`, `photo_urls text[]`, `dropoff_signature_url`, `email_sent`). Application code syncs them, not DB triggers. Drift possible.
- `auto_seed_inspection` trigger exists on `package_scan_logs` AFTER INSERT/UPDATE OF `matched_estimate_id` — creates `inspection_requests` row.
- Received components stored as strings (`"(2) Bracelet"`), not FK. No `components` enum or table.
- Photos go to public `attachments` Supabase storage bucket. Delivered to clients via direct `<img src>` links in HTML email, not attachments.

### Methodology lessons

**Apply messages must restate the surgical framing.** Bare "approved" leaves room for Lovable to interpret broadly. Convention established silo 8:
- Original prompt: `# Mode: Surgical fix — plan first, apply after I confirm`
- Apply trigger: `# Mode: Surgical fix — apply the plan exactly as shown` + restated constraints

**Don't proactively suggest stopping.** Mike explicitly pushed back twice. The "you're tired, save it for morning" framing is annoying, condescending, reduces trust. Trust the user to manage their own time. Only mention rest if the user raises it first.

**Don't anchor to a time without checking.** Claude anchored to "11:43pm hard stop at 2am" and kept referencing those times into Mike's afternoon (4:08pm est). Sessions can pause and resume; don't assume continuity of time.

**When Lovable says "fully patched," ask for ALL writers.** Silo 7's "verified working" claim on B1+B2 was technically true but incomplete. Follow-up audit silo 8 found the 12 other writers via explicit "list every place that writes to this table" prompting.

**Three-AI workflow is load-bearing, not optional.** A week of productive shipping has confirmed it. Lovable for live codebase + DB access; Claude for planning + cross-checking confident claims; Mike as deliberate bus.

---

## 3. What was deferred

### Designed, scoped, NOT yet applied

**Plan D — bump_job_to_in_progress trigger.** SECURITY DEFINER trigger function on `job_components`. When any component transitions INTO `in_progress`, bumps parent `jobs.status` to `in_progress` (only from intake/inspection/waiting_approval/in_queue/uncased; never downgrades), fires `net.http_post` to `rollisuite-status-push` with hardcoded anon Bearer (matches existing pattern in `set_job_status`, `assign_watchmaker`, `sync_watch_to_rollisuite`). Verified pg_net 0.19.5 available; no existing triggers on job_components to coordinate with; auth.uid() reimplementation justified.

Backfill decision: **skip the 4 candidates** (all in `waiting_approval` with `bracelet:in_progress` — EST-24800, 24185, 24392, 24187). Mike rejected pushing them to in_progress in production. Trigger going forward IS allowed to bump from waiting_approval (Mike OK'd future activity, just not the backfill).

Surgical-fix prompt not yet drafted. Touches core state machine; deserves its own focused apply session.

**Plan E — Shove to Ship Station.** Admin-only super override. New `src/components/sales/ForceShipStationDialog.tsx` cloned from `PaymentBypassDialog`. New `DropdownMenuItem` adjacent to existing "Push to Ship Station" at SalesOrderDetailPage.tsx:2093, preceded by separator, gated `{isAdmin && ...}`, destructive styling, ShieldAlert icon, label "Force to Ship Station (Override)".

On confirm:
1. UPDATE `sales_orders`: `is_paid=true, paid_at=now(), fulfillment_channel='ship'` (do NOT touch `balance_due`, `status`, `tracking_number`)
2. INSERT to `payment_bypass_log`: `sales_order_id`, `so_number`, `customer_id`, `bypass_amount=balance_due`, `balance_due`, `station='ship_station'`, `bypassed_by=auth.uid()`, `bypassed_by_email`, `notes=<reason>`
3. Invoke `send-security-alert` (verbatim from PaymentBypassDialog)

Required reason via Textarea, 10-char min enforced via disabled state on confirm button. Single AlertDialog (no two-step — no precedent in codebase). Visual badge on overridden Ship Station rows deferred to Plan E.2.

Surgical-fix prompt drafted silo 8, not yet pasted to RS Lovable.

**Fix B — handleQuickBarcodeSubmit re-entry guard.** One line: `if (isQuickSearching) return;` at top of handler in `src/pages/SalesOrderDetailPage.tsx`. Stops in-batch duplicate pattern (Write 2 on SO 501694: same part inserted twice on rapid Enter mashing during ~600ms Supabase lookup window).

Touched twice (silo 7, silo 8) but never landed. Silo 8 attempt: prompt sent to RW Lovable by mistake; RW correctly refused ("does not exist in this project"). Still needs RS-Lovable paste.

**Deep-link from JobHistoryLookup to Shop Floor.** Researched, verified ShopFloor.tsx has zero URL param support today. Identified ~6-10 line addition: `useSearchParams()` to read `?estimate=`, default `view` to `'lookup'`, fire existing `search()` on mount. Plus button + handler in JobHistoryLookup using existing `useNavigate` + `onOpenChange(false)` pattern (template at line 283 already in file).

Lower priority after pills shipped — pills may make the deep-link unnecessary.

### Open backlog items surfaced silo 8

- **SO 501718 QBO Customer Memo length error.** ValidationFault from QBO: `String length supplied 1,016, Max 1,000`. Source of 16-char overage unknown. Blocking SO 501718 push today.
- **QBO webhook CloudEvents migration deadline May 15, 2026 (today).** Reference: https://medium.com/intuitdev/upcoming-change-to-webhooks-payload-structure-2a87dab642d0. Whether RS migrated is unknown. If not, payment-status-back-from-QBO will fail silently.
- **2 candidate stuck SOs (55153, 55309).** Old enough (Jan 2026) to bypass cron's 21-row window. Part 2 button will catch them on next click. Confirm real-paid vs real-unpaid.
- **Premature `status='fulfilled'` source bug (Part 3 from SO 500950).** Something flips ship-channel orders to fulfilled without a shipment. Likely Pickup Station or fulfill-action path firing on ship channel. Separate investigation.
- **Push to Ship Station downgrade branch is destructive.** Lines 2075-2081 wipe shipped_qty / tracking_number / ship_date with no confirmation when current status is `shipped`. Worth adding confirm.
- **station_id propagation to all 14+ writers.** Real project, not urgent. Department fallback works fine for current consumer (pills).
- **commitDrop / handleCommit unification in ShopFloor.tsx.** Tech debt.

---

## 4. What was wrong

### Theories that broke under verification

**"Job History Lookup needs station_id."** Silo 7's recommended current-state-block design assumed station_id would be populated for components going through shop floor. Silo 8 audit found 221/222 NULL. Design held only because Lovable's silo 7 spec also centered department fallback. Pills as shipped don't read station_id at all.

**"Silo 7's B1+B2 fully patched the writers."** Technically true (both ShopFloor.tsx handlers write station_id correctly). Misleading because 12+ other writers exist that silo 7 didn't audit. The "verified working" claim led Mike to expect station_id would populate over time; data showed otherwise.

**"There must be one writer that's broken."** Initial hypothesis on the NULL station_id finding. Wrong — both patched writers work; the explanation is unpatched writers dominate.

### False summits

**Job History Lookup design "ready to ship" after silo 7 investigation.** Silo 8 verification surfaced the station_id adoption finding that reframed the design (department fallback as primary path, not transitional). Pills shipped with that reframing — different design than silo 7's spec.

**"Just one more ship" (Fix B) at 11:43pm.** Plan was straightforward, prompt drafted, but pasted to wrong AI. RW Lovable correctly refused. Fix B still didn't ship silo 8.

### Wrong frame, corrected mid-session

**Current-state block design (per-component card stack) → pills.** Claude proposed Lovable's original current-state-block layout. Mike steered to "update the existing pills instead" — much simpler, fit the existing visual language. Shipped same session.

### Time on dead ends

- Repeated "you're tired, save it for morning" framing from Claude after Mike had already started doing work. Mike pushed back twice. Claude continued anyway briefly, then stopped after explicit "stop doing this please."
- Claude anchored to 11:43pm hard stop, kept referencing into Mike's afternoon. Wasted at least one message.
- Initial confused suggestion that a Lovable response about the Plan D trigger was "the pills apply." It was a different feature entirely. Mike corrected.

---

## 5. Key decisions

See `decisions/` folder for individual decision docs. Summary:

- **Pills design over current-state-block.** Tighter, fit existing visual language, no station_id resolution needed.
- **Fallback precedence: initials → department → nothing.** Initials match bulk-assign UI elsewhere; department titlecased as primary fallback given 99.5% NULL station_id.
- **SO 500950 split: Part 1 (data) + Part 2 (button widening); Part 3 (premature-fulfilled) deferred.** Unblock immediately, prevent future, separate the source-bug investigation.
- **Sync QBO button cap 25 calls, oldest-first, unpaid+qboInvoiceId only.** Same order of magnitude as old 15 cap.
- **Shove gating: admin-only (RS terminology).** Owner-only over owner+manager. Co-sign rejected as overkill / nonexistent feature.
- **Shove audit: reuse payment_bypass_log with station='ship_station'.** Reuse fuzzes semantics slightly, new table is cleaner but requires migration + RLS. Reuse picked for v1.
- **Shove writes: is_paid + paid_at + fulfillment_channel only. Leave balance_due.** AR still sees what's owed.
- **Shove reason: free text, 10-char min.** No structured dropdown until patterns emerge.
- **Plan D: trigger over frontend gating.** Frontend would only patch ShopFloor+BandKanban; trigger covers all 14+ writers.
- **Plan D: reimplement set_job_status logic, not reuse RPC.** auth.uid() checks fail for service-role writers.
- **Plan D backfill: skip 4 candidates.** Don't push waiting_approval jobs to in_progress in production.

---

## 6. Test fixtures / diagnostic anchors

See `fixtures/` folder for individual fixture docs. Summary:

- **SO 500950** — Ship Station visibility canonical case. 44 days old, QBO paid, locally unpaid until silo 8 manual fix.
- **SO 501694** — SO duplicate-line forensic test target (silo 7 carried forward). 28 lines / 16 distinct / 12 dups.
- **SO 501718** — QBO Customer Memo length error blocker.
- **EST-24392 (Larissa Oprysk)** — Single-bracelet test bed, JV-assigned, the 1/222 component with station_id populated.
- **EST-24800, 24185, 24392, 24187** — Plan D backfill candidates (all skipped).
- **SO 55153, 55309** — Stuck-SO candidates for verifying Part 2 button fix.

---

## 7. Useful SQL from silo 8

```sql
-- Action-type breakdown for station_id NULL diagnosis
SELECT action,
       COUNT(*) AS total,
       COUNT(station_id) AS with_station_id,
       COUNT(*) - COUNT(station_id) AS null_station_id
FROM job_component_logs
GROUP BY action
ORDER BY total DESC;
-- Result silo 8: status_change 90/6/84, backfill 42/0/42,
--                assigned 3/0/3, created 1/0/1
```

```sql
-- Stuck SO candidates beyond cron 21-row window
SELECT so_number, order_date, qbo_invoice_id
FROM sales_orders
WHERE qbo_invoice_id IS NOT NULL
  AND is_paid = false
  AND order_date >= now() - interval '120 days'
  AND order_date < now() - interval '21 days'
ORDER BY order_date ASC;
-- Result silo 8: 2 rows — 55153 (62451 / 2026-01-22),
--                          55309 (62599 / 2026-01-29)
```

```sql
-- Plan D backfill blast radius
SELECT j.status AS current_job_status,
       COUNT(DISTINCT j.id) AS job_count
FROM public.jobs j
WHERE j.status IN ('intake','inspection','waiting_approval','in_queue','uncased')
  AND EXISTS (
    SELECT 1 FROM public.job_components c
    WHERE c.job_id = j.id AND c.status = 'in_progress'
  )
GROUP BY j.status
ORDER BY job_count DESC;
-- Result silo 8: waiting_approval = 4 (only category;
--                all 4 bracelet:in_progress)
```

```sql
-- Audit any specific writer's station_id habits
-- (template — substitute table + filter)
SELECT created_at, station_id, action, notes
FROM job_component_logs
ORDER BY created_at DESC
LIMIT 30;
```
