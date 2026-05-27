---
silo: 9
date: 2026-05-25
status: decided — implementation in 2-3 month rebuild starting in ~2 weeks
last_validated: 2026-05-25
---

# Decision: RS + RW + RC Share One Supabase Project; M3KE Stays Separate

## Context

Current architecture: three separate Supabase projects (RS, RW, RC each), data synced via HTTP push and edge functions. M3KE is fourth, knowledge service on Fly.io with own Supabase.

Symptoms requiring fix:
- RS → RW intake push is 18-field caret string; lossy, schema mismatches
- Duplicate jobs from customer-mismatch on push
- Backfill operations running constantly to compensate for receiver field gaps
- "Trickle-down" data flow (estimate → drop-off → receive-watch → shop floor → customer portal) requires sync everywhere

## Options considered

**Option A: Keep 3 separate Supabase projects, fix the push pipe**
- Pros: No migration risk; clean separation of concerns
- Cons: Sync semantics remain a constant problem; backfill remains; data drift inevitable

**Option B: Merge RS + RW + RC into one Supabase**
- Pros: Single source of truth; no push pipe; backfill goes away; queries cross-app naturally
- Cons: Migration risk; auth model changes; RLS becomes more complex

**Option C: Merge all four (include M3KE)**
- Pros: Maximum unification
- Cons: M3KE has different workload (embeddings, Slack); access patterns conflict with operator/customer apps

## Decision

**Option B** — RS + RW + RC share one Supabase. M3KE stays separate.

## Canonical project

**RS Supabase (`djbjwcoddddywkgljuja`) becomes canonical.**

Rationale:
- Has the reference data (`model_references`, `parts` with curated brand/bracelet_model, `customers`, `estimates`)
- M3KE already points at RS via `ROLLISUITE_SUPABASE_URL` — no retargeting
- RC's `rc-timeline-status` edge function already lives on RS
- RW data is more component-flow, easier to migrate onto RS than vice versa

## What stays the same

- RS at rollisuite.com remains operator/owner-facing
- RW at rolliworks.com remains shop-floor-facing  
- RC at rolliconnect.com remains customer-facing
- Three separate Lovable projects, three separate deploys, three separate UIs
- M3KE stays on its own Supabase + Fly.io infrastructure
- Just one database underneath RS+RW+RC

## Open architectural decisions (deferred to 2-week planning phase)

- Auth model: one login across apps, or separate logins same DB?
- Migration approach: big bang nightly cutover, parallel writing, or read-only freeze?
- Does RC also share the merged Supabase, or stay separate via edge functions?
- Table naming/schema organization: prefixed (`crm_*`, `repair_*`) vs Postgres schemas vs current names?
- RLS strategy for shared DB with different operator/owner/customer permissions

## Conditions that would change the decision

- If Supabase capacity becomes a constraint at scale (currently very far from limit)
- If app-specific data isolation becomes a hard regulatory requirement
- If three Lovable projects can't reliably coordinate against a shared DB schema

## Tables to migrate from RW onto RS

Approximate list (15-25 tables):
- `jobs`
- `job_components`
- `job_component_logs`
- `watchmakers`
- `dept_technicians`
- Station/component metadata tables

## Evidence

- Silo 9 audit identified push-pipe as root cause of backfill burden
- Beacon investigation showed cross-app data flow gaps
- Reuse-first principle (one source of truth) consistent across silo 9 conversations
