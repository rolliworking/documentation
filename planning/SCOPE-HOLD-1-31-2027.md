# Scope Hold — 1/31/2027 Facility Cutover

> **Status:** Draft — requires Michael confirmation via SPEC-PREP-005/006 answers.
> **Purpose:** Explicit IN/OUT line for the January 2027 software deadline. Point here when scope creep pressure arises.
> **Enforcement:** D-023 Transferability Test — every IN item must pass or have documented waiver.

---

## Deadline framing

| Milestone | Date | Meaning |
|-----------|------|---------|
| Software sprint end | **1/31/2027** | Core rebuild slices complete, UAT-ready |
| Dry-run target | **1/15/2027** | Parallel run at current location |
| Facility opening | **~Feb 2027** | Physical move — software must work on day one |
| Michael home-based | **Mid-July 2026** | Role-based approvals must work before this date |

---

## IN scope for facility opening (draft — confirm in SPEC-PREP-005)

These items are **required** for safe shop operations at the new facility:

### Intake and inventory (P0)

| Item | Decision / discovery source | SPEC |
|------|----------------------------|------|
| Three-page intake: Shipping Receive, Drop-off, Receive Watch | D-019 amendment | SPEC-001 |
| Component/piece creation at Receive Watch only | D-020, A-20260629-007 | SPEC-001 |
| RW push after labels printed (idempotent overwrite) | A-20260629-008 | SPEC-001 |
| Watch + piece data model (X of Y) | D-022 | SPEC-002 |
| Possession without estimate path + stagnation alert | Q-015-A (pending answer) | SPEC-001 |

### Shop floor and custody (P0–P1)

| Item | Decision / discovery source | SPEC |
|------|----------------------------|------|
| Unified Shop Floor with touch drag-and-drop | W-37, Q-014-A (pending) | SPEC-005 / SPEC-007 |
| Safe bins as station_id namespace | A-20260629-009 | SPEC-002 |
| Visual inventory dashboard (watches by customer) | W-33 | SPEC-005 |
| Piece movement audit log | D-015, D-020 | SPEC-002 |
| QR bench workflow for status changes | D-016 | SPEC-005 |

### Photos and pickup (P1)

| Item | Decision / discovery source | SPEC |
|------|----------------------------|------|
| R2 for new photos + shared.intake_photos index | D-021 | SPEC-002 |
| Pickup before/after verification (soft ack) | A-20260629-006, W-38 | SPEC-003 |
| Custody release on pickup complete | Q-006, A-015 | SPEC-003 |

### Customer-facing (P1)

| Item | Decision / discovery source | SPEC |
|------|----------------------------|------|
| RC conversation reuse / dedupe | Q-012-A (pending) | SPEC-004 |
| Email reply routing to RC threads | Q-007-A (pending) | SPEC-006 |

### Platform (P0)

| Item | Decision / discovery source | SPEC |
|------|----------------------------|------|
| Shared Supabase schemas (rs, rw, rc, shared) | D-003 | SPEC-002 |
| Role-based auth (not named individuals) | D-023, Q-010 | SPEC-005 |
| Observable QBO sync (no silent failure) | D-020, PROD-FIX-001 | SPEC-009 |
| Cross-app integration contracts documented | Q-008, ROLLITIME-INTEGRATION | SPEC-010 |

### Chain of custody — software layer (P1, hardware parallel)

| Item | Decision / discovery source | SPEC |
|------|----------------------------|------|
| Label scan → audit log event | D-015 | SPEC-008 |
| Camera integration (positions TBD) | FACILITY-INTEGRATION-CHECKLIST | SPEC-008 |

---

## OUT of scope for facility opening (draft — confirm in SPEC-PREP-006)

**Explicitly deferred to POST-FACILITY-ROADMAP-Q2-Q4-2027.md:**

| Item | Reason deferred |
|------|-----------------|
| **Jarvis** AI operations assistant | Greenfield; no operational dependency for opening |
| **Authenticator V1** | Requires RolliCurator + dial corpus scale |
| **RolliCurator** full wizard | Starts Q2 2027; reference seed can be manual initially |
| **PartsWiki** | D-001 Path A deferred |
| **RolliShop** multi-location launch | Multiplication Test — needs facility #1 stable first |
| **M3KE chatbot in RS** | Auth pattern not locked; M3KE v1 standalone sufficient |
| **W-40** AI warranty reports | Needs RT integration + test table |
| **Historical photo migration** | D-021 — index backfill only, not file moves |
| **Paper carbon work order digitization** | A-009, Q-013 — out of scope |
| **Full RolliTime API integration** | Default OUT per SPEC-PREP-012 unless Michael marks IN |
| **Offline shop floor mode** | FACILITY-INTEGRATION F-34 — likely OUT |
| **PO automatic QBO sync** | A-20260629-002 — manual bridge preserved |
| **Variance notifier W-41** | P1 — may slip to Q2 if SPEC-001 time-constrained |
| **Full reference data reconciliation** | REBUILD-PREREQUISITES #2 — stub acceptable if RolliCurator follows |

---

## Thesis clarification — AI module split

The **7-month sprint to 1/31/2027** is about **operational transferability** (D-023), not shipping every AI module.

| Category | Facility opening | Post-facility |
|----------|------------------|---------------|
| Operational apps (RS, RW, RC) | IN — must work without Michael daily | Harden + optimize |
| Data capture (photos, tests, audit) | IN — seams must exist | AI consumes captured data |
| AI modules (Jarvis, Authenticator, M3KE-in-RS) | OUT | Q2–Q4 2027+ |
| Knowledge capture (RolliCurator) | OUT (manual expertise OK short-term) | Q2 2027 start |

**Jarvis split:** Jarvis is not a facility-opening requirement. Daily hit list (W-21) ships as **structured RS to-do feature** without LLM if needed. Jarvis attaches later when `ai_modules.*` auth and RolliCurator dataset exist.

---

## Scope creep response script

When a new request arrives during SPEC weekend:

1. Is it on the **IN** list above? → Schedule in current SPEC.
2. Is it on the **OUT** list? → "Deferred per SCOPE-HOLD; see POST-FACILITY-ROADMAP."
3. Is it neither? → Add to WISHLIST; default **OUT** unless Michael elevates with written IN justification.
4. Does it fail Transferability Test? → Redesign or waiver (TRANSFERABILITY-WAIVERS.md).

---

## Confirmation required

Michael: mark each IN item **confirm** or **cut** in SPEC-PREP-005/006 answers. This doc updates to match.

---

_End of scope hold. Living document — update when SPEC-PREP answers land._
