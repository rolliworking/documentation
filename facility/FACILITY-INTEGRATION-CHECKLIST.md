# Facility Integration Checklist — Software Meets Physical Layout

> **Status:** Questionnaire — not decisions. Answer during facility planning and SPEC-PREP snapshot review.
> **Purpose:** Enumerate every hybrid decision where software behavior depends on physical shop layout.
> **Companion:** `QUESTIONS-2026-06-30-spec-prep.docx` (SPEC-PREP-007 through SPEC-PREP-011)

---

## How to use this checklist

Each section lists **questions to answer**, not prescriptive answers. When Michael answers SPEC-PREP items, update this doc with decisions and link to HUMAN-ANSWERS entries.

---

## 1. Floor plan and station mapping

| # | Question | Software impact | Answered? |
|---|----------|-----------------|-----------|
| F-01 | Where is the receiving bench relative to the safe and customer counter? | Stage 1a shipping receive workflow, label printer placement | ☐ |
| F-02 | How many watchmaker benches in the new facility? | `shared.stations` row count, QR code print run | ☐ |
| F-03 | Does each bench get a dedicated iPad mount or shared pool? | RW auth sessions, device registration | ☐ |
| F-04 | Is there a separate polishing room with its own stations? | `department` metadata on stations (D-022) | ☐ |
| F-05 | Where is the testing / RolliTime station? | RT scan trigger, status push to RW | ☐ |
| F-06 | Should software station IDs match printed labels on the floor? | W-37 drag-and-drop, staff training | ☐ |
| F-07 | How many safe bins / slots inside the vault? | `safe_bin_A1` … naming scheme (A-20260629-009) | ☐ |

---

## 2. Safe and vault integration

| # | Question | Software impact | Answered? |
|---|----------|-----------------|-----------|
| F-08 | Scan at safe door only, or scan per bin slot? | Piece movement granularity, audit log density | ☐ |
| F-09 | Who performs bulk safe audit (weekly inventory count)? | W-33 visual inventory, role assignment | ☐ |
| F-10 | Does the safe have interior camera coverage? | D-015 chain-of-custody footage query | ☐ |
| F-11 | Is the vault separate from the workshop safe? | Additional station_ids, custody_status values | ☐ |
| F-12 | UPS / power backup for safe area network gear? | D-020 — silent failure if network drops mid-scan | ☐ |

---

## 3. IP cameras and chain of custody (D-015)

| # | Question | Software impact | Answered? |
|---|----------|-----------------|-----------|
| F-13 | Camera at receiving bench? | Label scan → frame lookup at intake | ☐ |
| F-14 | Camera at pickup counter? | W-38 before/after verification evidence | ☐ |
| F-15 | Camera inside safe? | Theft deterrence, discrepancy resolution | ☐ |
| F-16 | Continuous recording vs snapshot-on-scan? | Storage cost, retention policy | ☐ |
| F-17 | Retention period (days)? | NVR config, legal/compliance | ☐ |
| F-18 | Who can view footage (roles)? | Auth model for camera API | ☐ |
| F-19 | Camera vendor (Nest, Reolink, NVR, other)? | Integration API for SPEC-008 | ☐ |

---

## 4. Kiosks and customer-facing touchpoints

| # | Question | Software impact | Answered? |
|---|----------|-----------------|-----------|
| F-20 | Customer kiosk at pickup only, or also lobby? | RC portal vs kiosk app scope | ☐ |
| F-21 | Kiosk authentication (customer login vs staff-assisted)? | RC auth flow | ☐ |
| F-22 | Payment on kiosk or counter-only? | QBO payment link, pickup bypass policy | ☐ |
| F-23 | Kiosk screen size and orientation? | UI layout for W-38 photo comparison | ☐ |

---

## 5. iPad and bench hardware

| # | Question | Software impact | Answered? |
|---|----------|-----------------|-----------|
| F-24 | Fixed mount height and angle at each bench? | QR scan reliability (D-016) | ☐ |
| F-25 | Bluetooth barcode scanner vs camera-only? | RW scan UX, external scanner API | ☐ |
| F-26 | Shared iPads vs assigned per watchmaker? | Staff_id on piece custody | ☐ |
| F-27 | Voice-to-text for job notes (W-30)? | iOS permissions, RW notes table | ☐ |
| F-28 | Shop Floor on iPad — retire Component Status? (Q-014-A) | Single unified floor UI | ☐ |

---

## 6. Network, power, and IT ownership

| # | Question | Software impact | Answered? |
|---|----------|-----------------|-----------|
| F-29 | Ethernet to each fixed station vs Wi-Fi only? | Latency for drag-and-drop, photo upload | ☐ |
| F-30 | PoE switch for cameras? | Cable plan, VLAN design | ☐ |
| F-31 | Separate VLAN for shop floor devices? | Security, guest Wi-Fi isolation | ☐ |
| F-32 | Who installs network (contractor vs Michael)? | Timeline risk for SPEC-PREP-003 | ☐ |
| F-33 | Label printer model and connection (USB vs network)? | RS receive watch label wizard | ☐ |
| F-34 | Failover if internet down — can shop floor operate offline? | Offline mode scope (likely OUT) | ☐ |

---

## 7. Signage and physical labels

| # | Question | Software impact | Answered? |
|---|----------|-----------------|-----------|
| F-35 | QR codes on bench for process actions (D-016)? | Print run, RW process QR table | ☐ |
| F-36 | Physical watch labels — same barcode as software? | Scan consistency across RS/RW | ☐ |
| F-37 | Color coding on floor (departments)? | May mirror `department` on stations | ☐ |

---

## 8. Move timeline coordination

| # | Question | Software impact | Answered? |
|---|----------|-----------------|-----------|
| F-38 | Parallel run old + new software before move? | Cutover strategy (SCOPE-HOLD) | ☐ |
| F-39 | Hardware install complete before software UAT? | Testing window | ☐ |
| F-40 | Staff training days scheduled before opening? | Runbook readiness (D-023) | ☐ |

---

## Decision log (fill as answers arrive)

| Date | ID | Decision | Source |
|------|-----|----------|--------|
| — | — | — | — |

---

_End of checklist. Questions only until facility planning sessions complete._
