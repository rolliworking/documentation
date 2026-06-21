# Rolliworks — Rebuild Prerequisites

**Purpose:** Every architectural decision and data migration that must be true *before* any module rebuild starts. The rebuild can't safely begin until these are resolved.

**One-line goal anchor:** _[paste the rebuild one-line goal here]_

**How to use:** Each item has a status. When all items are ✅, the rebuild has a stable foundation to build against.

---

## Prerequisite checklist

| # | Item | Status | Owner |
|---|---|---|---|
| 1 | Naming collision resolved (RolliSuite vs. RepairShopr vs. "RS") | ⬜ | Michael |
| 2 | Reference data consolidated — RS as single source of truth | ⬜ | Michael + watchmaker supervisor |
| 3 | AI module authentication pattern defined | ⬜ | Michael (with AI advisor) |
| 4 | Photo library access control defined | ⬜ | Michael |
| 5 | Schema boundary mapping done (rs.* / rw.* / rc.* / shared.* / ai_modules.*) | ⬜ | Michael (with AI advisor) |
| 6 | Cross-app contract documentation done (real payloads, not doc payloads) | ⬜ | Qwen drafts, Opus refines |
| 7 | Cursor READMEs complete for RC, RW, RS | ⬜ | Qwen drafts, Opus refines |

---

## #1 — Naming collision resolved

**Problem:** "RS" can mean RolliSuite (the ERP) or RepairShopr (the third-party SaaS). Team and docs use the same shorthand for both.

**Output:** A one-line glossary, committed to every doc:
- RolliSuite = the ERP (we built)
- RepairShopr = the third-party SaaS (we use)
- Never say "RS" alone

**Effort:** 5 minutes.

---

## #2 — Reference data consolidated (RS as source of truth)

**Problem:** Today, RS and RW each maintain their own caliber and reference-model libraries. They've drifted. Watchmakers can be working against the wrong tolerances.

**Decision (made):** RS is the source of truth. All apps — RW, RolliTime, future modules — read caliber and reference-model data from RS.

**Output:**
- Compare RS and RW reference libraries (SQL query, run in both Supabase projects)
- Identify drift (where RW has data RS doesn't, or where the same caliber has different specs)
- Reconcile drift into RS as the authoritative version
- Migrate the data into shared schema during the rebuild (`shared.calibers`, `shared.reference_models`)
- Point RW, RolliTime, and future modules at the shared tables
- Decommission RW's local copy

**Why this is high priority:**
- Live operational risk today (quality, pricing accuracy)
- Blocks RolliTime (RolliTime's §4.2 contract assumes this is in place)
- Blocks every future module that needs caliber context (M3KE chatbot, Jarvis, watch tools, authenticator)

**Effort:** Comparison query — 30 min to draft. Run + review — 1-2 hours. Reconciliation — depends on drift size, likely 1-3 days.

---

## #3 — AI module authentication pattern

**Problem:** M3KE, RolliTime, Jarvis, and future modules all need to read from and/or write to the shared database. No defined pattern exists for how they authenticate.

**Decision needed:** Authentication model. Likely answer: per-module service role with scoped read on `rs.*` / `rw.*` / `rc.*`, scoped write on each module's own schema under `ai_modules.*`.

**Output:** A short document defining the auth pattern, with example policies and a checklist for adding a new AI module.

**Effort:** 1-2 hours to define and document.

**Reference:** Cross-cutting Q1 + Q3 in `FUTURE-INTEGRATIONS.md`.

---

## #4 — Photo library access control

**Problem:** RolliTime owns the dial-photo library. RS, the authenticator app, and future modules need to read from it. Auth model is currently [OPEN] in the RolliTime contract (§6.4).

**Decision needed:** API key model, per-consumer permissions, who can read what.

**Output:** Documented access policy added to `ROLLITIME-INTEGRATION.md` §6.4.

**Effort:** 1 hour to define.

**Likely outcome:** Same pattern as #3 — module-scoped service roles.

---

## #5 — Schema boundary mapping

**Problem:** The shared-Supabase rebuild requires deciding which tables live where. Without this, the rebuild has no map.

**Output:** A table-by-table document listing every table and its target schema:
- `rs.*` — RolliSuite-owned tables
- `rw.*` — RolliWorking-owned tables
- `rc.*` — RolliConnect-owned tables
- `shared.*` — cross-app reference data (customers, watches, parts, calibers, reference_models)
- `ai_modules.*` — AI module outputs (M3KE, Jarvis, etc.)

**Effort:** Depends on the Cursor READMEs (#7) being done first. Once those exist, schema boundary mapping is ~1 day of Opus work.

**Reference:** Cross-cutting question in `FUTURE-INTEGRATIONS.md` and the shared-Supabase architecture discussion.

---

## #6 — Cross-app contract documentation

**Problem:** Documented RS→RW intake contract is not the real contract (hard-coded fields, missing payload fields, no audit log). Actual behavior diverges from documented behavior.

**Output:** Every cross-app HTTP call, webhook, and shared token documented with its *real* payload — observed from code, not from old docs.

**Effort:** Qwen drafts from edge function code, Opus refines, 2-3 days.

---

## #7 — Cursor READMEs complete for RC, RW, RS

**Problem:** No factual current-state documentation exists for the three apps. Every AI conversation has to rediscover the system.

**Output:** Three READMEs, one per app, generated from actual code per the existing Cursor README prompt. Refined and committed to docs repo.

**Effort:** Qwen drafts on Corsair (1-2 hours per app), Opus refines (1 hour per app), 2-3 days total.

---

## Order of work

Items #1, #3, #4 are decisions (fast — hours).
Items #6, #7 are documentation runs (Qwen + Opus pipeline — days).
Items #2, #5 depend on the documentation existing first.

Suggested sequence:
1. Day 1: #1 (naming) + start workflow map
2. Days 2-3: #7 (Cursor READMEs)
3. Day 4: #6 (cross-app contracts) + #3 (auth pattern) + #4 (photo access)
4. Day 5: #5 (schema boundary mapping)
5. Day 6+: #2 (reference data comparison, reconciliation, consolidation)

Then rebuild proper begins.
