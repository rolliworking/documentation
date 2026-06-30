# Q-014 — Inventory RolliWorking iPad UI + gestures

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-29
**Depends on:** Q-002
**Human answers applied:** A-20260628-007 (non-developer staff; visual GUI only), W-37 (Shop Floor drag-and-drop), D-016 (QR bench workflow)
**Inputs read:**
- `apps/rw/src/config/routes.tsx` (47 routes)
- `apps/rw/src/pages/wm/` (WmRoom, WmPartsApprovals)
- `apps/rw/src/pages/ShopFloor.tsx`, `WorkQueue.tsx`, `StationScanner.tsx`, `ComponentStatus.tsx`
- `apps/rw/src/components/scanner/BarcodeScanner.tsx`
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**RW iPad usability audit** — Catalog every RolliWorking screen, primary actions, touch vs mouse patterns, and QR replacement candidates for D-016. Foundation for W-37 Shop Floor GUI on iPad.

---

## 2. iPad posture summary

| Tier | Routes | Posture |
|------|--------|---------|
| **iPad-native** | `/wm`, `/wm/approvals` | WmShell, touch gestures, PWA manifest, safe-area |
| **Wedge-friendly** | Shop Floor assign tab, StationScanner, BulkAssign, PossessionScan | Bluetooth scanner; no camera required |
| **Desktop-first** | All AppShell pages (35+) | Sidebar nav, hover states, small `<select>` controls |

**Only `/wm*` is intentionally iPad-built.** Everything else is a sidebar web app used on iPad via browser.

---

## 3. Page inventory (staff-facing, by frequency)

### Tier A — Daily bench/floor

| Page | Route | Primary action | Est. taps / friction | QR candidate |
|------|-------|----------------|----------------------|--------------|
| **WmRoom** | `/wm` | Advance/downgrade status; scan; parts | 2–3 tap per status change | **P1** — replace chevrons with process QR (D-016) |
| **WorkQueue** | `/work-queue` | Status `<select>`; WM assign; email | **5–7 taps** status change | **P0** — scan ref then process QR |
| **ShopFloor** | `/shop-floor` | Assign: tap station + scan; Lookup: **mouse drag** | Drag blocked on touch | **P0** — W-37 touch drag |
| **StationScanner** | `/station-scanner` | Label scan → process QR | 2 scans (D-016 prototype) | **Already QR** |
| **ComponentStatus** | `/component-status` | Morning/evening scan batches; HTML drag | Drag mouse-only | Merge into W-37 |
| **BulkAssign** | `/bulk-assign` | Scan tech + jobs | Wedge OK | Low — already scan-first |
| **PartsRequest** | `/parts-request` | Build parts request | Dense form | Medium |

### Tier B — Intake / lookup

| Page | Route | Friction |
|------|-------|----------|
| NewInspection | `/inspections/new` | Long form; camera + wedge |
| ComponentLookup | `/component-lookup` | Wedge OK; read-only |
| Customers | `/clients` | Wedge import; no camera button on iPad landscape |

### Tier C — Admin/reports

Analytics, Users, Wiki, Email Templates, etc. — desktop tables; rare iPad use.

---

## 4. W-37 drag-and-drop candidates

| Screen | Current drag | Touch status | W-37 priority |
|--------|-------------|--------------|---------------|
| **ShopFloor lookup** | SVG pointer drag; `station_id` commit | **Blocked** — toast "use a mouse" | **#1** — primary W-37 surface |
| **ComponentStatus** | HTML5 drag to safe | Mouse only | **#2** — merge or deprecate |
| **WorkQueue** | None (dropdown moves) | N/A | Replace with drag or QR |
| **WmRoom** | None | Touch OK | Status via QR not drag |

```407:415:apps/rw/src/pages/ShopFloor.tsx
    if (e.pointerType === "touch") {
      toast.error("Drag on touch isn't supported yet — use a mouse.");
      return;
```

**W-37 rebuild requirements (from A-007):** Visual drag between station columns; safe as drop target; exceptions (cancel → safe; safe → queue); no code/status menus for staff.

---

## 5. QR scan patterns today

| Mechanism | Coverage |
|-----------|----------|
| `useBarcodeScannerInput` (wedge) | WorkQueue, ShopFloor, BulkAssign, StationScanner, most floor pages |
| `BarcodeScanner` (camera) | WmRoom, PartsRequest; WorkQueue **only if width <768px** (iPad landscape often excluded) |
| Process QRs (`STATUS:`, `DEPT:`, `SAFE:`) | StationScanner only |
| Tech badge QRs | BulkAssignSettings print |

**D-016 gap:** Bench workflow still uses chevrons/dropdowns — not scan-first.

---

## 6. Voice-to-text

| Surface | Present? |
|---------|----------|
| Job notes / comments | **No** (W-30) |
| AiAssistant chat | Yes — Web Speech API only |
| WmRoom / inspection | **No** |

---

## 7. QR transformation priority (D-016)

| Rank | Screen | Rationale |
|------|--------|-----------|
| 1 | WorkQueue status changes | Highest daily tap count; Michael pain |
| 2 | ShopFloor W-37 touch drag | Unblocks floor ops on iPad |
| 3 | WmRoom status chevrons | Already on iPad; swap to process QR |
| 4 | ComponentStatus | Consolidate into ShopFloor |

---

## 8. Open questions

### Q-014-A: Deprecate ComponentStatus in favor of unified ShopFloor?

**Type:** scope
**Default:** Yes — single floor UI per W-37; ComponentStatus scan batches become Shop Floor modes.

---

## 9. Acceptance criteria check

- ✅ Every routed page named with primary action
- ✅ Tap-count estimates for common workflows
- ✅ QR candidates ranked
- ✅ W-37 touch drag gap documented with line evidence
- ✅ A-007 non-developer constraint applied

---

_End of discovery. Q-014 complete._
