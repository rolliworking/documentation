---
silo: 8
date: 2026-05-15
status: designed, not yet applied
last_validated: 2026-05-15
---

# Decision — Shove to Ship Station: admin-only, reuse payment_bypass_log, free-text reason

## Question

How should the admin override that releases a stuck SO into Ship Station Ready state be designed?

## Sub-decisions

### Naming: "Shove" vs "Force"

Final label in the dropdown: "Force to Ship Station (Override)". But the operational term in Mike's vocabulary is "Shove" — a push routes, a shove forces. The handoff doc captures both: "Push routes, Shove forces" as the mental model; "Force to Ship Station (Override)" as the UI label.

### Gating: admin-only

**Considered:** admin-only, admin + manager, co-sign (admin initiates, manager approves), per-key permission.

**Chose:** admin-only via `useAuth().isAdmin`.

**Why:**
- Tightest control. No ambiguity about accountability.
- Co-sign (two-person integrity) is an order-of-magnitude bigger feature (pending-action queue, notification, approval/reject UI, expiry). Doesn't exist in the app anywhere.
- Per-key permission was rejected — the whole point of a super override is that it's not delegable.

**Note:** RS has no `owner` role — `admin` IS the super-user. RW has owner/manager/staff; admin doesn't exist there. Always check which app when discussing roles.

### Audit table: reuse `payment_bypass_log`

**Considered:** Reuse `payment_bypass_log` with `station='ship_station'`, OR create new `sales_order_overrides` table.

**Chose:** Reuse with `station='ship_station'`.

**Why:**
- Existing table already has the shape: `sales_order_id, so_number, customer_id, bypass_amount, balance_due, station, bypassed_by, bypassed_by_email, notes, created_at`.
- RLS already configured: any auth can insert, only admins can read.
- SMS alert via `send-security-alert` already wired in `PaymentBypassDialog`.
- No migration, no new RLS, no new admin-readable surface.

**Trade-off accepted:** Semantic blur. "Payment bypass" and "Force to Ship Station" are conceptually distinct but use the same row shape. Mitigated by `station='ship_station'` being filterable. If override usage becomes regular, migrate to dedicated table later.

### Field writes: is_paid + paid_at + fulfillment_channel only

**Considered:** Also clear `balance_due` (matches what QBO sync does when invoice is paid).

**Chose:** Leave `balance_due` alone.

**Why:**
- `is_paid=true` is sufficient to pass the queue's isPaid gate.
- Preserving `balance_due` means AR / staff can see "this was a force-release, the actual balance will reconcile when QBO syncs."
- Less destructive: if the override turns out to be wrong, balance_due is intact.

### Reason: required free text, 10-char min

**Considered:** Optional, required free text, required structured dropdown ("QBO sync stuck" / "Manual external payment" / "Comp shipment" / "Other").

**Chose:** Required free text, 10-char min enforced via disabled state on confirm button.

**Why:**
- Required because audit value is the point — overrides without reasons are noise.
- Free text rather than dropdown because the categories aren't known yet. Let patterns emerge in the log; convert to dropdown later if needed.
- 10-char min is light fence against accidental empty / single-word reasons.

### Confirmation: single AlertDialog (no two-step)

**Considered:** Single modal vs two-step ("Are you sure?" after the first modal).

**Chose:** Single AlertDialog with destructive copy.

**Why:**
- No two-step precedent in the codebase. `PaymentBypassDialog` uses single AlertDialog with prominent destructive copy + SMS alert as the friction. That pattern works; reuse it.
- Two-step adds friction without commensurate safety benefit when SMS alert already lands.

### Reversibility: no undo

**Considered:** Add an "Undo override" action that reverts is_paid + paid_at.

**Chose:** No undo. Fix forward if needed.

**Why:**
- Undo is just another manual UPDATE.
- "Fix forward" matches the rest of the system's philosophy.
- Adding undo invites the question of who can undo (presumably also admin) which doesn't add safety since the original admin could just as well manually UPDATE the row.

### Placement: same dropdown, adjacent to Push

**Considered:** Same dropdown as Push, separate dropdown/menu entirely, buried in settings/gear submenu.

**Chose:** Same dropdown, immediately after "Push to Ship Station" (line 2093), preceded by `<DropdownMenuSeparator />`, destructive styling.

**Why:**
- Discoverable when admin needs it (during a stuck-SO incident, time matters).
- Adjacent to Push so the relationship is obvious.
- Destructive styling + admin gate prevent fat-finger.
- Burying in a settings submenu would hide it too much.

### Visual badge on overridden rows: deferred

Visual badge in Ship Station list (`PaidUnshippedOrdersList`) to mark overridden rows is real but deferred to Plan E.2. Requires LEFT JOIN against `payment_bypass_log`. Doesn't block v1; can ship after the action lands.

## What would change these decisions

- If override usage exceeds ~weekly: migrate audit to dedicated `sales_order_overrides` table with cleaner semantics.
- If admins routinely need to override on behalf of someone else (delegation): revisit per-key permission.
- If false-override mistakes happen: revisit undo.
