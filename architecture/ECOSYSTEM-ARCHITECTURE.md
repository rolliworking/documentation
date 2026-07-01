# Rolliworks Ecosystem Architecture

> **Status:** Living reference — synthesized from DECISIONS-REGISTRY (D-001 through D-023), discovery phase, integration contracts, and NORTH-STAR.md.
> **Note:** `PLANNED-CODEBASE.md` is referenced in D-019 but does not exist on disk yet; this document serves as the interim canonical architecture view until that file is authored.

---

## 1. One-line goal

Build an integrated watch-service-center suite where fixing operational bottlenecks captures valuable byproducts and reusing that data improves the next workflow — with one canonical data model, auditable workflows, and reduced dependence on any single individual (D-023 Transferability Test).

---

## 2. App roster and language assignments (D-017)

| Abbr | App | Language | Database | Role |
|------|-----|----------|----------|------|
| RS | RolliSuite | TypeScript / React | Shared Supabase (target) | ERP, client files, source of truth for reference data |
| RW | RolliWorking | TypeScript / React | Shared Supabase (target) | Watchmaker workflow, shop floor, bench iPad |
| RC | RolliConnect | TypeScript / React | Shared Supabase (target) | Customer portal, messaging, estimates |
| RT | RolliTime | Python / FastAPI | **Own DB** (per D-008) | Testing, dial-photo library |
| AA | Authenticator | Python | TBD | Genuine vs counterfeit (deferred post-facility) |
| — | M3KE | Python | `ai_modules.*` | Knowledge / chatbot (deferred) |
| — | Jarvis | Python | `ai_modules.*` | Operations assistant (deferred) |
| — | RolliCurator | Python | `ai_modules.*` | Expertise capture wizard (deferred) |

**Today:** RS, RW, RC run on **separate Supabase projects** with fragmented auth (Q-010). **Target:** D-003 shared project, schemas `rs.*`, `rw.*`, `rc.*`, `shared.*`, `ai_modules.*`.

---

## 3. High-level data flow

```
                    ┌─────────────────────────────────────────────────────────┐
                    │              CUSTOMERS / STAFF / INTUIT QBO               │
                    └────────────┬───────────────────────────────┬────────────┘
                                 │                               │
         ┌───────────────────────▼──────────┐    ┌───────────────▼──────────────┐
         │         RolliConnect (RC)         │    │      QuickBooks Online        │
         │  portal · estimates · messaging   │    │  customers · invoices · bills │
         └───────────────────────┬──────────┘    └───────────────┬──────────────┘
                                 │                               │
                                 │         ┌─────────────────────┘
                                 │         │ qbo-* edge functions
                                 ▼         ▼
         ┌───────────────────────────────────────────────────────────────┐
         │                    RolliSuite (RS)                             │
         │  intake (3-page) · estimates · SO · pickup · QBO · inspection  │
         └───────┬───────────────────────────────┬───────────────────────┘
                 │                               │
                 │ rollisuite-intake (sync only) │ shared.* reads
                 ▼                               ▼
         ┌───────────────────┐         ┌────────────────────────────────┐
         │ RolliWorking (RW) │◄───────►│ shared schema (target)         │
         │ shop floor · bench│         │ customers · watches · pieces     │
         └─────────┬─────────┘         │ intake_photos · audit_log · staff│
                   │                   └────────────────┬───────────────┘
                   │ API contracts                      │
                   ▼                                    ▼
         ┌───────────────────┐              ┌─────────────────────┐
         │ RolliTime (RT)    │              │ ai_modules.*        │
         │ own PostgreSQL    │              │ M3KE · Jarvis ·     │
         │ testing · dials   │              │ RolliCurator (future)│
         └───────────────────┘              └─────────────────────┘
```

---

## 4. Schema ownership map (target state)

| Schema | Owner writes | Primary consumers | Key tables (proposed) |
|--------|--------------|-------------------|------------------------|
| `shared.*` | RS (admin), cross-app via RLS | RS, RW, RC, RT (read) | `customers`, `watches`, `pieces`, `staff`, `intake_photos`, `audit_log`, `stations` |
| `shared.*` reference | RS + RolliCurator promotion | RW, RT, AA | `calibers`, `reference_models`, `bracelets`, `dials` (D-002, D-011) |
| `rs.*` | RS app | RS staff | estimates, sales_orders, possession (Stage 1), invoices mirror |
| `rw.*` | RW app | Watchmakers | job status, bench notes, shop floor events |
| `rc.*` | RC app | Customers, Vianna | conversations, portal sessions, intake forms |
| `ai_modules.*` | Python modules | RS UI (read), analytics | per-module output tables |
| RT DB | RolliTime | RS, RW, AA (API) | test sessions, dial library (D-008) |

**Identity spine (D-022):** `shared.watches` (one row per serial) + `shared.pieces` (X of Y assembly state). Supersedes parallel `client_property` vs `job_components` location models.

**Photo spine (D-021):** `shared.intake_photos` index bridges R2 (new), Supabase Storage (historical), RT dial library.

---

## 5. Integration contract summaries

### RolliTime ↔ RS / RW (`ROLLITIME-INTEGRATION.md`, D-008)

| Direction | Payload | Auth | Status |
|-----------|---------|------|--------|
| RT → RW | Status "in testing" on scan | API key | Intent only — Q-008 |
| RT → RS | Test results + photos on client file | API key | Intent only |
| RS → RT | Caliber / reference data (authoritative) | API key | CSV seed today; live pull target |
| RW → RT | Pull test data for watchmaker analytics | API key | Out of RT push scope |

**Rebuild:** New `shared.timing_test_results` (A-20260629-003); 15-minute poll + on-demand at scan (A-20260629-004).

### RS ↔ RW (intake boundary)

| Event | Today | Target (D-019, D-020) |
|-------|-------|------------------------|
| Stage 1 possession | Drop-off + Shipping Receive | `rs.intake_possession` (auditable) |
| Stage 2 commit | Receive Watch | Creates `shared.pieces`; pushes RW after labels printed (A-20260629-008) |
| Legacy `rollisuite-intake` | Creates `job_components` from estimate flow | **Sync/notify only** — not creator |

### RS ↔ QBO (`Q-005` discovery)

| Path | Trigger | Telemetry |
|------|---------|-----------|
| Customer pull | Daily cron 06:00 UTC | `qbo_sync_log` — **broken** (stuck started) |
| Invoice pull | Hourly :15 UTC | `qbo_sync_log` — **broken** (PROD-FIX-001) |
| Customer/invoice push | SO fulfill (manual) | Operator-visible toasts |
| PO receive | Manual | No sync log |

### RC ↔ RS (portal intake)

| Event | Today | Target |
|-------|-------|--------|
| Web form submit | New conversation always | Reuse open thread? (Q-012-A open) |
| Email replies | help@ inbox | Route to RC thread? (Q-007-A open) |

---

## 6. Foundational patterns

### Two-stage intake (D-019)

```
Stage 1a: Shipping Receive ──┐
Stage 1b: Drop-off ──────────┼──► possession record (in building, unverified)
                             │
Stage 2: Receive Watch ──────┴──► verify + create pieces + audit log
```

### Silent failure banned (D-020)

Every cron, webhook, and edge function: durable log row at start, explicit success/fail at end, non-200 on log failure.

### Transferability Test (D-023)

Every SPEC must pass five questions (Absence, Mike, Onboarding, Handoff, Multiplication) before lock.

### Chain of custody (D-015)

Label scan → camera frame query → audit log. Depends on W-33 inventory + station_id authority (W-37).

---

## 7. Deployment topology (target)

```
┌─────────────────────────────────────────────────────────────┐
│ Supabase (shared project)                                    │
│  PostgreSQL · Auth · Storage (legacy photos) · Edge Functions │
│  pg_cron: qbo-*, maintenance, rw-status, vault, gold price   │
└─────────────────────────────────────────────────────────────┘
         ▲              ▲              ▲
         │              │              │
    RS (Vite)      RW (Vite)      RC (Vite)
    Lovable/       iPad web       portal
    Emergent

┌──────────────────┐     ┌──────────────────┐
│ RolliTime        │     │ Cloudflare R2    │
│ Python/FastAPI   │     │ new photos       │
│ separate host    │     │ (D-021)          │
└──────────────────┘     └──────────────────┘
```

**Open decision:** Same Supabase project vs new project (SPEC-PREP-001).

---

## 8. SPEC build order (from discovery digest)

| Priority | SPEC | Decisions |
|----------|------|-----------|
| P0 | Two-stage intake | D-019, D-020, D-022 |
| P0 | Shared schema + auth | D-003, D-010, Q-010 |
| P0 | QBO observable sync | D-020, PROD-FIX-001 |
| P1 | Shop floor W-37 | D-022, Q-014 |
| P1 | Photo index W-38 | D-021 |
| P1 | RC client matching | Q-012 |
| P1 | RT integration | Q-008, D-008 |

---

## 9. Companion documents

| Document | Purpose |
|----------|---------|
| `DECISIONS-REGISTRY.md` | All D-* decisions |
| `DRAFT-CANONICAL-DATA-MODEL.md` | SPEC-002 starting point |
| `SCOPE-HOLD-1-31-2027.md` | IN/OUT for facility cutover |
| `TRANSFERABILITY-TEST.md` | SPEC design gate |
| `ROLLITIME-INTEGRATION.md` | RT contracts |
| `REBUILD-PREREQUISITES.md` | Pre-build checklist |
| Discovery `Q-001` through `Q-015` | Evidence base |

---

_End of ecosystem architecture. Update when hosting decision (SPEC-PREP-001) locks._
