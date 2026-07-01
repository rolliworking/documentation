# Q-011 — Audit component registration (unblocks Shop Floor)

**Status:** complete
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-28
**Depends on:** Q-001 (intake map), Q-004 (Shop Floor symptoms)
**Human answers applied:** A-20260628-002 (derive from actual components, not itemType), A-20260628-004 (job_components schema with optional identifiers per type)
**Inputs read:**
- `apps/rs/src/pages/intake/ClientWatchEntryPage.tsx` — saveMutation L1202–1242
- `apps/rs/src/pages/intake/ReceivePackagesPage.tsx` — package receive save (notes tags)
- `apps/rs/src/components/labels/IntakeLabelWizard.tsx` — RW push L502–536
- `apps/rs/src/hooks/useRolliworkingSync.ts` — caret payload L97–141
- `apps/rw/supabase/functions/rollisuite-intake/index.ts` — flow + component INSERT L570–616
- `apps/rw/src/pages/ShopFloor.tsx` — component lookup dependency
- `apps/rw/src/lib/component-creation.ts` — backfill `createComponents` L244–304
- `apps/rw/supabase/migrations/20260402224644_*.sql` — `job_components` table
- `documentation/discovery/Q-001-receive-watch-module.md`, `Q-004-shop-floor-audit.md`
- `documentation/known-gaps/markerless-estimates.md`, `receiver-undersized-component-insert.md`
- **Skill:** `mapping-legacy-workflows`

---

## 1. Workflow name and purpose

**Component registration** — The process by which physical watch parts received at intake become structured `job_components` rows that Shop Floor, Work Queue, and station routing can display and move.

Mike's operational quote: *"components aren't registered so shop floor won't work."*

This audit maps **where registration should happen**, **where it actually happens**, and **the specific gap** between staff-confirmed received components and RW `job_components` rows.

---

## 2. Trigger

| Event | Should register components? | Actually creates `job_components`? |
|-------|----------------------------|-----------------------------------|
| RS drop-off save (`ClientWatchEntryPage`) | Yes — staff toggles `receivedComponents` | **No** — writes `client_property` + notes tags only |
| RS package receive save (`ReceivePackagesPage`) | Yes — staff taps received items | **No** — writes `package_scan_logs` notes only |
| Label wizard complete (`IntakeLabelWizard.handleDone`) | Yes — push to RW with received data | **Indirect** — fires `sendIntakeToRolliworking` |
| RW `rollisuite-intake` edge function | Yes — create rows from intake | **Yes** — but from **estimate flow**, not received toggles |
| Shop Floor BackfillPanel | Last resort manual fix | **Yes** — manual INSERT via UI |

**Critical path:** Component rows are created only when the label wizard completes and RW intake runs — not when RS intake saves.

---

## 3. Actors

| Actor | Role |
|-------|------|
| Staff (intake) | Confirms `receivedComponents` toggles or package received items |
| `ClientWatchEntryPage` | Persists RS-side receipt to `client_property` |
| `IntakeLabelWizard` | Triggers RW sync on fresh intake label completion |
| `sendIntakeToRolliworking` | Caret-delimited POST to RW `rollisuite-intake` |
| `rollisuite-intake` | Creates `jobs` + `job_components` from estimate-derived **flow** |
| `get_job_profile` RPC | Computes flow from estimate markers (markerless → `BAND_ONLY_FLOW`) |
| Shop Floor | Reads `job_components`; shows "No components" when empty |
| BackfillPanel | Manual component creation when registration failed |

---

## 4. Inputs

### RS intake — what staff confirms

| Input | UI location | Persisted where | Used for `job_components`? |
|-------|-------------|-----------------|---------------------------|
| `receivedComponents[]` | ClientWatchEntry toggles | `client_property.notes` as `[received:…]` | **No** |
| `itemType` radio | ClientWatchEntry | Drives `received_bracelet`/`received_case` booleans | **No** (wrong derivation per A-20260628-002) |
| `packageReceivedItems` | ReceivePackages buttons | `notes` / scan log | **No** |
| Service codes / dept flags | Estimate + intake | RS columns + RW push | Indirect (flow derivation) |

### RW intake payload (caret positions 10–18)

| Field | Sent by RS? | Parsed by RW? | Used for component INSERT? |
|-------|------------|---------------|---------------------------|
| `received_bracelet` (pos 10) | Yes | JSON path yes; **caret path skipped** → defaults `true` | Flow nudge only (~L571) |
| `received_case` (pos 11) | Yes | Same | Flow nudge only |
| `received_components` (pos 18) | Yes (`sendIntakeToRolliworking` L121) | Parsed to `payloadReceivedComponents` L549–564 | **Stored on `jobs`; NOT used for row creation** |
| `flow` | Sometimes | Overrides if present | **Determines which INSERT block runs** |

---

## 5. Steps (registration flow)

### Intended workflow (operator mental model)

1. Staff receives watch/components at intake
2. Staff confirms what physically arrived (`receivedComponents`)
3. System registers each component as a trackable entity
4. Shop Floor shows colored dots per component
5. Techs drag/scan components through stations

### Actual workflow (code)

**Step A — RS save (no RW call)**

```1206:1227:apps/rs/src/pages/intake/ClientWatchEntryPage.tsx
      const derivedReceivedBracelet = itemType === 'watch';
      const derivedReceivedCase = itemType === 'watch' || itemType === 'watch_head_only';
      // ...
          notes: [
            formData.notes,
            receivedComponents.length > 0
              ? `[received:${receivedComponents.join(',')}]`
              : null,
          ].filter(Boolean).join(' ') || null,
          received_bracelet: derivedReceivedBracelet,
          received_case: derivedReceivedCase,
```

No `sendIntakeToRolliworking` on save. No `job_components` write.

**Step B — Label wizard → RW push (only registration trigger)**

```517:536:apps/rs/src/components/labels/IntakeLabelWizard.tsx
    if (isFreshIntake && shouldSync) {
      sendIntakeToRolliworking({
        // ...
        receivedBracelet,
        receivedCase: effectiveReceivedCase,
        receivedComponents: customerData.receivedComponents || [],
      });
```

Fire-and-forget; failures silent (`useRolliworkingSync.ts` L137–140).

**Step C — RW derives flow (not from received toggles)**

```570:573:apps/rw/supabase/functions/rollisuite-intake/index.ts
    const flow = payloadFlow
      || (hasWatchmaker && hasBandWork && receivedBracelet ? 'SPLIT_FLOW'
        : hasWatchmaker                                    ? 'STANDARD_FLOW'
        :                                                    'BAND_ONLY_FLOW');
```

`hasWatchmaker` / `hasBandWork` come from estimate service codes via `get_job_profile` — **not** from `[received:…]` tags.

**Step D — Component INSERT by flow**

```593:616:apps/rw/supabase/functions/rollisuite-intake/index.ts
    if (!existingComponents?.length && jobId && labelBase) {
      if (flow === 'STANDARD_FLOW') {
        await supabase.from('job_components').insert([
          { job_id: jobId, component_type: 'watch_head', ... },
          { job_id: jobId, component_type: 'case', ... },
        ]);
      }
      if (flow === 'SPLIT_FLOW') { /* + bracelet */ }
      if (flow === 'BAND_ONLY_FLOW') {
        await supabase.from('job_components').insert([
          { job_id: jobId, component_type: 'bracelet', ... },
        ]);
      }
    }
```

`payloadReceivedComponents` is parsed (L549–564) but **never referenced** in INSERT logic.

**Step E — Shop Floor reads result**

If `job_components.length === 0` or wrong types → "No components on this job" + BackfillPanel.

---

## 6. Outputs

| Layer | Output | Registration quality |
|-------|--------|---------------------|
| RS `client_property` | Row with notes tags, booleans | Captures staff intent; unstructured |
| RS `parts` (CW-) | Scannable part number | Inventory barcode; not Shop Floor components |
| RW `jobs` | Job row with flow, `received_components` text[] | Flow often wrong; received_components unused |
| RW `job_components` | 1–3 rows by flow | **The Shop Floor spine** — wrong/missing when flow wrong |
| Shop Floor UI | Colored dots / empty state | Downstream symptom |

---

## 7. Cross-app touches

| Touch | Direction | Notes |
|-------|-----------|-------|
| RS → RW | `rollisuite-intake` POST | Only via label wizard; not on RS save |
| RW Shop Floor | Reads `job_components` | Never reads `jobs.received_components` |
| RolliConnect | Timeline from `get_job_profile.flow` | Wrong flow → wrong customer-facing stages |
| Q-004 Shop Floor audit | Symptom layer | This doc is root-cause layer |
| D-019 two-stage intake | Architectural target | Possession (Stage 1) vs verification/commit (Stage 2) not enforced today |

---

## 8. Edge cases

| Edge case | Registration result |
|-----------|---------------------|
| Markerless estimate (~10% post-rollout) | `BAND_ONLY_FLOW` → bracelet-only row for watchmaker job |
| Package receive without ClientWatchEntry + labels | No RW push → **zero components** |
| Label wizard skipped / reprint only (`isFreshIntake=false`) | No RW push |
| RW push fails silently | Staff sees success; Shop Floor empty |
| Duplicate jobs path (intake L431–434) | Components on orphan job row |
| Partial pickup / band-only custody split | `[split_custody:…]` in notes only; no component model |
| Multi-item intake (husband/wife components) | Single notes tag; no per-item IDs (A-20260628-002 gap) |

---

## 9. Workarounds observed

| Workaround | Where | Why it exists |
|------------|-------|---------------|
| `[received:…]` text in notes | `client_property.notes` | No structured component table at RS |
| `itemType`-derived booleans | ClientWatchEntry saveMutation | Shortcut instead of component toggles (bug per A-002) |
| BackfillPanel | ShopFloor.tsx ~1481–1674 | Manual component creation when intake failed |
| ComponentStatus silent backfill | RW frontend useEffect | Compensates for NULL department/station_id |
| `createComponents` on scan | component-creation.ts | Creates missing rows at station assign time |
| Marker enforcement at estimate save | silo 10 proposed fix | Prevents wrong flow at source |

---

## 10. Open questions

### Q-011-A: Should staff's component checklist drive what gets registered in the shop system?

**Type:** scope
**Question:** When staff check off what physically arrived (watch head, bracelet, case), should that checklist directly create the shop-floor tracking records — instead of the system guessing from the estimate's department markers?
**Why it matters:** Today the shop floor often shows the wrong components because the system ignores what staff actually confirmed and instead follows estimate markers that are wrong about 10% of the time.
**What I observed:** Staff selections are saved as text notes on the customer asset record. The shop system receives the data but creates component records based on estimate flow, not the checklist. *(Technical: received_components parsed but unused in rollisuite-intake.)*
**My best guess:** In the rebuild, Receive Watch creates components from the checklist (D-020). For the legacy app, adding checklist-driven creation is a medium-risk hotfix after the markerless estimate fix.
**Default if no answer in 7 days:** Rebuild uses Receive Watch per D-020; legacy hotfix adds mapping after markerless flow fix.

### Q-011-B: Should saving the Receive Watch page immediately notify the shop system?

**Type:** architectural
**Question:** Today, the shop system only learns about a watch when the label wizard finishes — not when staff save the Receive Watch page. Should saving Receive Watch immediately send the watch to the shop queue, or should we keep waiting until labels are printed?
**Why it matters:** The two-stage intake pattern says possession (package receive) and verification (Receive Watch) are separate steps. Delaying shop notification until label printing creates a gap where staff think the watch is registered but the shop floor sees nothing.
**What I observed:** Receive Watch save writes to RolliSuite only. The shop notification fires from the label wizard completion step. *(Technical: sendIntakeToRolliworking only from IntakeLabelWizard.)*
**My best guess:** Rebuild triggers shop notification at Stage 2 verification commit; legacy keeps label-wizard trigger with documented risk.
**Default if no answer in 7 days:** Keep label-wizard trigger for legacy; document as coupling risk until D-019 rebuild.

---

## 11. Schema summary

### RS — no component registration table

| Storage | Format | Shop Floor readable? |
|---------|--------|---------------------|
| `client_property.notes` | `[received:complete_watch,bracelet]` | **No** |
| `client_property.received_bracelet/case` | BOOLEAN | RW flow nudge only |
| `client_property_components` | — | **Does not exist** |

### RW — `job_components` (Shop Floor spine)

| Column | Required at INSERT | Set by intake? |
|--------|-------------------|----------------|
| `job_id` | Yes | Yes |
| `component_type` | Yes | Yes (`watch_head`, `case`, `bracelet`) |
| `label_id` | Yes | Yes |
| `status` | Yes | `'in_queue'` |
| `department` | No | Yes (Fix B) |
| `station_id` | No | **Usually NULL** |
| `assignee_*` | No | **Usually NULL** |

Per A-20260628-004 rebuild target: add `name` (required), `serial_number`, `model_number`, `date_code`, `notes` — **not in current schema**.

### RW — `jobs.received_components`

`text[]` — populated from intake payload, **never read by Shop Floor UI**.

---

## 12. Registration points: intended vs actual

| # | Intended | Actual | Gap |
|---|----------|--------|-----|
| 1 | Staff toggles → structured component records | Text in `notes` | No RS component table |
| 2 | Intake save registers components | Save writes RS only | **No RW call on save** |
| 3 | RW push uses received toggles | Push uses estimate **flow** | Wrong types for ~10% markerless |
| 4 | `received_components` payload → rows | Parsed but ignored | **Specific code gap** |
| 5 | Package path registers components | Notes only | No RW push from package alone |
| 6 | Full row for Shop Floor | 4–5 of 17 columns on INSERT | NULL `station_id` on 99.5% |

**The specific gap (acceptance criterion):**

> Row creation in `job_components` is driven by `flow` derived from `get_job_profile` estimate markers (L570–573), **not** from staff-confirmed `receivedComponents`. When `get_job_profile` returns `BAND_ONLY_FLOW` for a markerless watchmaker estimate, `rollisuite-intake` inserts only `{ component_type: 'bracelet' }` (L611–615) even though staff may have confirmed `complete_watch` in `[received:…]` notes. Shop Floor then shows no watch-head dot.

---

## 13. "25-day component creation outage" post-mortem

**No dedicated post-mortem document exists** in the repo (TASK-QUEUE referenced "prior chats").

**Best reconstruction** from `markerless-estimates.md` and silo 9/10:

| Period | Event |
|--------|--------|
| 2026-05-02 | Service markers introduced on estimate line items |
| 2026-05-02 → ~2026-05-27 | ~25 days where markerless estimates silently → `BAND_ONLY_FLOW` |
| Steady-state post-rollout | ~10% of new estimates still markerless |
| Impact | **Mis-registration**, not zero registration — wrong component types created |
| Pre–Fix B (~2026-05-25) | INSERTs also omitted `department`, forcing backfill culture |

This is the operational "outage": Shop Floor appeared broken because watch-head/case rows were missing for genuine watchmaker jobs, not because INSERT code was disabled.

---

## 14. Minimal change proposal (fix what's broken)

Priority order — scoped fixes, not full rebuild:

| P | Fix | File | Effort | Impact |
|---|-----|------|--------|--------|
| **P0** | Enforce `[X]` markers at estimate save OR fix `get_job_profile` ELSE branch | RS SQL / estimate UI | ~2 hr | Stops wrong flow at source |
| **P1** | Map `payloadReceivedComponents` → component INSERT types | `rollisuite-intake/index.ts` after L564 | ~20 lines | Aligns registration with staff toggles (A-002) |
| **P1** | Set `station_id: 'lock_pre_queue'` on every intake INSERT | `rollisuite-intake` L595–614 | ~3 lines | Fixes dot placement |
| **P2** | Parse caret positions 10/11 for `receivedBracelet`/`receivedCase` | intake parser | ~10 lines | Stops default-true override |
| **P2** | Surface RW push failure toast (not silent) | `useRolliworkingSync.ts` L137 | ~5 lines | Operator visibility |
| **P3** | SQL backfill `station_id` on existing NULL rows | one-time migration | ops | Cleans historical data |

**Out of scope for minimal fix:** New `client_property_components` table, D-019 two-stage schema, Shop Floor rewrite.

**Rebuild alignment (A-20260628-004):** Extend `job_components` with `name`, optional `serial_number`/`model_number`/`date_code` per component type when registration is rebuilt.

---

## 15. Comparison to workflow docs / human answers

| Source | Finding |
|--------|---------|
| A-20260628-002 | Derive from actual components — **code still uses itemType + flow, not toggles** |
| A-20260628-003 / D-019 | Two-stage intake not enforced — RS save ≠ verification commit |
| A-20260628-004 | Schema needs optional identifiers — **current table lacks these columns** |
| Q-004 ROOT CAUSE 1 | Markerless → BAND_ONLY — **confirmed at rollisuite-intake L570–573** |
| Q-001 §7 | No job creation on intake save — **confirmed; RW push only at label wizard** |

---

## 16. What "done" means (acceptance criteria check)

- ✅ Every intended registration point named with file/line evidence
- ✅ Specific gap: `flow`-based INSERT ignores `receivedComponents`; markerless misclassification
- ✅ 25-day outage reconstructed (mis-registration window, not code disabled)
- ✅ Minimal-change proposal scoped to fix-not-rebuild
- ✅ A-20260628-002 and A-20260628-004 applied as rebuild constraints

---

_End of discovery. Q-011 complete._
