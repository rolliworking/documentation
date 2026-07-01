# Risk Register — 1/31/2027 Cutover

> **Status:** Living register — review weekly during SPEC/BUILD phase.
> **Scale:** Severity (1–5), Likelihood (1–5), Exposure = S × L (max 25).
> **Companion:** `SCOPE-HOLD-1-31-2027.md`, `TRANSFERABILITY-TEST.md`

---

## Risk summary matrix

| ID | Risk | S | L | Exp | Status |
|----|------|---|---|-----|--------|
| R-01 | SPEC scope creep | 5 | 4 | 20 | Open |
| R-02 | Operator burnout (Michael) | 5 | 3 | 15 | Open |
| R-03 | Business emergency diverts attention | 4 | 3 | 12 | Open |
| R-04 | Agent/tool output quality limits | 4 | 3 | 12 | Open |
| R-05 | Facility construction delay | 4 | 2 | 8 | Open |
| R-06 | Data migration failure / corruption | 5 | 3 | 15 | Open |
| R-07 | Staff not trained on new flows | 4 | 3 | 12 | Open |
| R-08 | Hosting decision delayed | 4 | 3 | 12 | Open |
| R-09 | RLS complexity blocks shared schema | 5 | 3 | 15 | Open |
| R-10 | Cutover failure — shop can't operate | 5 | 2 | 10 | Open |
| R-11 | PROD-FIX-001 not applied — QBO blind | 4 | 4 | 16 | Open |
| R-12 | Michael unavailable mid-July without role handoff | 5 | 3 | 15 | Open |
| R-13 | RolliTime integration underestimated | 3 | 4 | 12 | Open |
| R-14 | Photo index backfill incomplete | 3 | 3 | 9 | Open |
| R-15 | Lovable/Emergent deploy pipeline breaks | 4 | 2 | 8 | Open |
| R-16 | Cross-app auth migration breaks live apps | 5 | 2 | 10 | Open |
| R-17 | IP camera / facility IT not ready | 3 | 3 | 9 | Open |
| R-18 | QBO customer cron remains broken | 4 | 4 | 16 | Open |
| R-19 | Transferability Test failures ship anyway | 4 | 3 | 12 | Open |
| R-20 | Parallel repo / documentation drift | 3 | 3 | 9 | Open |

---

## Detailed risks

### R-01 — SPEC scope creep

| Field | Detail |
|-------|--------|
| **Description** | Each SPEC weekend adds "just one more" feature; 1/31/2027 deadline slips |
| **Severity** | 5 — misses facility opening |
| **Likelihood** | 4 — high pressure environment |
| **Mitigation** | SCOPE-HOLD doc; D-023 gate; default OUT; weekly scope review |
| **Early warning** | SPECs growing past 2-day estimate; IN list items added without Michael sign-off |
| **Owner** | Michael |

### R-02 — Operator burnout

| Field | Detail |
|-------|--------|
| **Description** | Michael is product owner + reviewer + facility planner + still running shop |
| **Severity** | 5 |
| **Likelihood** | 3 |
| **Mitigation** | SPEC-PREP-004 coverage matrix; defer OUT items; 72h question snapshot cadence not daily |
| **Early warning** | Answered question snapshots backlog > 2; SPEC reviews skipped |
| **Owner** | Michael |

### R-03 — Business emergency diverts attention

| Field | Detail |
|-------|--------|
| **Description** | Rush job, client crisis, vendor failure pulls Michael away for days |
| **Severity** | 4 |
| **Likelihood** | 3 |
| **Mitigation** | Transferability Test — systems run without Michael; documented escalation roles |
| **Early warning** | HUMAN-QUEUE unanswered > 7 days; PROD-FIX items aging |
| **Owner** | Michael |

### R-04 — Agent/tool output quality limits

| Field | Detail |
|-------|--------|
| **Description** | Cursor/Emergent produce plausible but wrong code; Michael can't verify at code level |
| **Severity** | 4 |
| **Likelihood** | 3 |
| **Mitigation** | Bounded task packets; reviewing-prs-governance skill; acceptance criteria per slice; SHA discipline |
| **Early warning** | "Build complete" without test evidence; governance review skipped |
| **Owner** | Opus review layer |

### R-05 — Facility construction delay

| Field | Detail |
|-------|--------|
| **Description** | Feb 2027 opening slips; software ready too early or crunch mis-timed |
| **Severity** | 4 |
| **Likelihood** | 2 |
| **Mitigation** | SPEC-PREP-003 firm date; software UAT at current location regardless |
| **Early warning** | No construction milestone updates in 30 days |
| **Owner** | Michael |

### R-06 — Data migration failure

| Field | Detail |
|-------|--------|
| **Description** | Customer/watch/piece migration corrupts or loses records |
| **Severity** | 5 |
| **Likelihood** | 3 |
| **Mitigation** | SPEC-PREP-002 cutoff strategy; dry-run migration; legacy read-only fallback |
| **Early warning** | Migration test shows > 1% unmatched rows |
| **Owner** | SPEC-002 author |

### R-07 — Staff not trained on new flows

| Field | Detail |
|-------|--------|
| **Description** | Vianna/watchmakers can't operate three-page intake or shop floor on opening day |
| **Severity** | 4 |
| **Likelihood** | 3 |
| **Mitigation** | SPEC-PREP-013 parallel run; role runbooks; D-023 Onboarding Test per SPEC |
| **Early warning** | No training scheduled 30 days before opening |
| **Owner** | Vianna + Michael |

### R-08 — Hosting decision delayed

| Field | Detail |
|-------|--------|
| **Description** | Same-project vs new-project undecided; blocks SPEC-002 migrations |
| **Severity** | 4 |
| **Likelihood** | 3 |
| **Mitigation** | SPEC-PREP-001 answer by SPEC weekend; default same-project |
| **Early warning** | SPEC-002 started without hosting decision recorded |
| **Owner** | Michael |

### R-09 — RLS complexity blocks shared schema

| Field | Detail |
|-------|--------|
| **Description** | Row-level security across rs/rw/rc/shared proves unmanageable; cutover stalls |
| **Severity** | 5 |
| **Likelihood** | 3 |
| **Mitigation** | Q-010 auth discovery; phased RLS; service role patterns for edge functions |
| **Early warning** | Auth SPEC exceeds 1 week; repeated RLS policy bugs in UAT |
| **Owner** | SPEC-005 author |

### R-10 — Cutover failure

| Field | Detail |
|-------|--------|
| **Description** | Big-bang switch breaks daily operations; rollback not possible |
| **Severity** | 5 |
| **Likelihood** | 2 |
| **Mitigation** | orchestrating-cutover-tests skill; parallel run; cherry-pick promotion per CLAUDE.md |
| **Early warning** | No rollback procedure in PACKET; cutover brief missing |
| **Owner** | Michael |

### R-11 — PROD-FIX-001 not applied

| Field | Detail |
|-------|--------|
| **Description** | Invoice sync runs 24×/day with zero telemetry indefinitely |
| **Severity** | 4 |
| **Likelihood** | 4 — draft ready but not applied |
| **Mitigation** | PROD-FIX-001-DRAFT.md; Michael review tonight; separate from rebuild |
| **Early warning** | Zero `sync_type=invoice` rows after 48h post-approval |
| **Owner** | Michael / Lovable |

### R-12 — Michael unavailable mid-July without handoff

| Field | Detail |
|-------|--------|
| **Description** | Home-based shift begins but approvals still route to Michael |
| **Severity** | 5 |
| **Likelihood** | 3 |
| **Mitigation** | SPEC-PREP-004; role-based auth in SPEC-005; SCOPE-HOLD IN list |
| **Early warning** | Any SPEC says "ask Michael" in approval path |
| **Owner** | Michael |

### R-13 — RolliTime integration underestimated

| Field | Detail |
|-------|--------|
| **Description** | RT assumed IN but API not built; testing workflow breaks at new facility |
| **Severity** | 3 |
| **Likelihood** | 4 |
| **Mitigation** | Default OUT per SPEC-PREP-012; manual workflow acceptable 90 days |
| **Early warning** | SPEC-010 scope expands beyond poll + shared table |
| **Owner** | Michael |

### R-14 — Photo index backfill incomplete

| Field | Detail |
|-------|--------|
| **Description** | W-38 pickup comparison can't find intake photos |
| **Severity** | 3 |
| **Likelihood** | 3 |
| **Mitigation** | D-021 — new photos indexed at capture; backfill script Q2 |
| **Early warning** | Pickup UAT shows missing before photos for recent jobs |
| **Owner** | SPEC-002 |

### R-15 — Deploy pipeline breaks

| Field | Detail |
|-------|--------|
| **Description** | Lovable or Emergent can't deploy shared-schema changes |
| **Severity** | 4 |
| **Likelihood** | 2 |
| **Mitigation** | Cursor surgical path for migrations; test branch discipline |
| **Early warning** | Failed deploy without rollback SHA |
| **Owner** | Michael |

### R-16 — Auth migration breaks live apps

| Field | Detail |
|-------|--------|
| **Description** | Merging Supabase projects logs out all users / breaks RC portal |
| **Severity** | 5 |
| **Likelihood** | 2 |
| **Mitigation** | Phased cutover; auth bridge period; Q-010 patterns |
| **Early warning** | Auth change without cutover brief |
| **Owner** | SPEC-005 |

### R-17 — Facility IT not ready

| Field | Detail |
|-------|--------|
| **Description** | Cameras, network, iPad mounts not installed when software UAT starts |
| **Severity** | 3 |
| **Likelihood** | 3 |
| **Mitigation** | FACILITY-INTEGRATION-CHECKLIST; software UAT at current location first |
| **Early warning** | F-29–F-34 unanswered 60 days before opening |
| **Owner** | Michael + contractor |

### R-18 — QBO customer cron remains broken

| Field | Detail |
|-------|--------|
| **Description** | 162+ stuck `started` rows; customer data stale |
| **Severity** | 4 |
| **Likelihood** | 4 — known since 2026-01-12 |
| **Mitigation** | Push-on-fulfill canonical (A-010); rebuild cron per D-020; separate from PROD-FIX-001 |
| **Early warning** | Stuck row count still growing |
| **Owner** | Rebuild SPEC-009 |

### R-19 — Transferability failures ship

| Field | Detail |
|-------|--------|
| **Description** | SPECs lock without passing D-023 tests |
| **Severity** | 4 |
| **Likelihood** | 3 |
| **Mitigation** | Transferability Test Results section mandatory; waiver tracker |
| **Early warning** | Empty Transferability section in 01-SPEC.md |
| **Owner** | Opus / SPEC author |

### R-20 — Documentation drift

| Field | Detail |
|-------|--------|
| **Description** | rebuild-hq, documentation, app repos diverge; agents work from stale truth |
| **Severity** | 3 |
| **Likelihood** | 3 |
| **Mitigation** | HUMAN-ANSWERS canonical; BUILD-LOG + DELTA per slice; SHA reporting |
| **Early warning** | Discovery files contradict HUMAN-ANSWERS |
| **Owner** | Cursor agents |

---

## Top 5 by exposure (act now)

1. **R-01** Scope creep → use SCOPE-HOLD
2. **R-11** PROD-FIX-001 → review tonight
3. **R-18** QBO customer cron → SPEC-009
4. **R-06** Data migration → SPEC-PREP-002 answer
5. **R-12** July handoff → SPEC-PREP-004 answer

---

## Review log

| Date | Reviewer | Notes |
|------|----------|-------|
| 2026-06-30 | Cursor Agent | Initial register created |

---

_End of risk register. Update exposure scores when mitigations land._
