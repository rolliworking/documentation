# RolliConnect (RC) — Documentation

> Documentation for RolliConnect, the customer CRM + portal layer of the Rolliworks ecosystem.

**Repo:** https://github.com/rolliworking/documentation (this content lives in `rc/`)  
**Production:** my.rolliworks.com  
**Supabase project:** `ibwrjsmuvrqtokoqpogu`  
**Last updated:** May 27, 2026

---

## What is RolliConnect?

RolliConnect (RC) is the customer-facing layer for Rolliworks Inc, a high-end watch and bracelet repair business (~17 service requests/day, ~$1.5M ARR). RC handles:

- **Customer portal** — clients track their repair status, view photos, approve estimates, message staff
- **Internal CRM inbox** — staff (Mike, Vienna) manage customer conversations
- **Email routing** — inbound replies via Postmark, outbound via Resend
- **Status display** — pulls workflow status from the workshop (RW) and ERP (RS)

RC is one of three connected systems. See `architecture/three-system-overview.md`.

---

## Repository structure

```
documentation/rc/
├── README.md                          ← you are here
├── architecture/                      ← system design, data flow, contracts
│   ├── three-system-overview.md
│   ├── token-strategy.md
│   └── email-routing.md
├── features/                          ← feature registry with status + entry points
│   └── feature-registry.md
├── security/                          ← auth, RLS, compliance
│   └── security-overview.md
├── technical-debt/                    ← known bugs, hacks, roadmap
│   ├── known-issues.md
│   └── roadmap.md
├── plans/                             ← scoped feature plans (not yet built)
├── prompts/                           ← reusable prompts for AI build tools
│   ├── builds/
│   ├── research/
│   └── diagnostics/
├── handoffs/                          ← session handoff documents
│   └── session-handoff-2026-05-27.md  ← latest state
├── handoff/current.md                 ← pointer to latest RC handoff (repo convention)
├── silos/                             ← RC silo closeouts (when split from root)
├── known-gaps/                        ← RC-only gaps (stub; see also technical-debt/)
├── decisions/                         ← RC-only decisions (stub)
└── fixtures/                          ← RC test fixtures (stub)
```

---

## Quick orientation for new contributors (human or AI)

1. **Start here:** `architecture/three-system-overview.md` — understand the 3-system topology
2. **Then:** `features/feature-registry.md` — what exists and where
3. **Then:** `technical-debt/known-issues.md` — what's broken or fragile
4. **For current state:** [handoffs/session-handoff-2026-05-27.md](handoffs/session-handoff-2026-05-27.md) — most recent working state

## Cross-repo docs (repo root)

- [Silo 9](../silos/silo-9.md) — RC timeline v2
- [Handoff: ecosystem](../handoff/current.md) — RS/RW/RC rolling handoff

---

## The three systems

| System | Name | Role | Supabase Project | Production URL |
|--------|------|------|------------------|----------------|
| **RS** | RolliSuite | ERP / intake / estimates / shipping | `djbjwcoddddywkgljuja` | rollisuite.com |
| **RW** | RolliWorking | Workshop / job management / shop floor | `pkgnrcfqrldwjibghefm` | rolliworking.lovable.app |
| **RC** | RolliConnect | Customer CRM + portal | `ibwrjsmuvrqtokoqpogu` | my.rolliworks.com |

All three are built on Lovable (React + Vite + TypeScript + Tailwind + Supabase).

---

## Build tool division of labor

- **Lovable** — new features, UI work, database schema, edge functions
- **Cursor** — surgical fixes, multi-file refactors, code verification, git operations, backup deploy path
- **Claude (chat)** — architecture decisions, cross-system coordination, prompt drafting, planning

---

## Documentation discipline

This is a living document. Maintenance protocol:

1. **Start of build session:** load relevant docs as context
2. **During session:** decisions get logged, conflicts flagged
3. **End of session:** update affected docs, write a handoff if state changed significantly
4. **Version control:** commit doc changes alongside or after code changes

See `prompts/` for the Master Knowledgebase maintenance prompt.

---

## Tech stack summary

- **Frontend:** React + Vite + TypeScript + Tailwind CSS
- **Backend:** Supabase (Postgres, Auth, Edge Functions, Storage)
- **Email inbound:** Postmark (reply.rolliworks.com)
- **Email outbound:** Resend (transactional)
- **Intake:** Wix forms → portal-intake-router
- **Hosting:** Lovable (frontend), Supabase (backend)
- **Source control:** GitHub (rolliworking org)
