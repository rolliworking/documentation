# Session Handoff — May 27, 2026

This document captures the current working state for the next person/AI picking up RC work.

---

## Production state RIGHT NOW

**my.rolliworks.com is WORKING.** It broke mid-session and was recovered.

What happened:
1. Internal notes + Mark Unread bump feature deployed via Lovable Publish
2. Lovable deployed frontend but did NOT run the migration → `is_internal` column missing
3. Every send + inbox query failed silently (no error toast, no message inserted, lost row colors)
4. Diagnosed by Cursor: schema/frontend mismatch
5. Migration run MANUALLY in Supabase SQL editor → production recovered
6. Color coding restored, sends working

---

## What needs immediate attention

### 1. Internal notes feature — ABANDON & CLEAN UP
The @mention internal-notes feature LEAKS staff comments to customers and must be removed.
- Vienna (logged in) sending `@mike ...` → reached the customer's email
- `@vienna ...` correctly stayed internal
- Root cause: team_users RLS likely restricts authenticated SELECT to own row, so Vienna's client can't see Mike's name to match
- DECISION: abandon the feature
- ACTION: disable `detectInternalMention()` (return false) OR full revert; KEEP the Mark Unread bump (it works)
- The `is_internal` column + RLS can stay (harmless)
- URGENT: until removed, staff must not use @mentions expecting privacy

### 2. Reply token display lost
The `<reply_token>@reply.rolliworks.com` line + Copy button under the client name disappeared (likely the Test↔main merge kept Test's ConversationPanel which lacked the block). Re-add it.

---

## What shipped today (needs verification)

- Shared team read state (one row per conversation, team-wide)
- Inbox case-insensitive team_users lookup (`ilike`)
- RS role allowlist fix — Vienna (manager) can Send Portal Invite
- Per-customer reply token (`clients.reply_token`, 271 backfilled)
- Reply token handler fixes (CC scan, staff attribution, loop prevention) — UNVERIFIED end-to-end
- Mark Unread → Conversations + bump to top (via last_message_at)

---

## In flight (other chats / tools)

- **Band flow split (Task 1)** — RW-owned, commit `98ce8a3` on RC `rc-timeline-status` (sectioned DTO v2). Customer should see correct flow (band omits In Testing, uses Polish/Clean).
- **RS→RW data integrity (Task 2)** — separate chat. Urgent missing field TBD; also service_type, date, auth, customer ID.
- **Inspection approval portal URL** — Cursor fixed `submit-inspection-approval` to look up RC by email (was using wrong ID key). Pending commit/push.
- **get-rc-portal-link** — verified working (200 with key, 401 without, graceful miss)

---

## Key reference data

| Thing | Value |
|-------|-------|
| RC Supabase | `ibwrjsmuvrqtokoqpogu` |
| RW Supabase | `pkgnrcfqrldwjibghefm` |
| RS Supabase | `djbjwcoddddywkgljuja` |
| Mike | mike@rolliworks.com, role admin, team_users id `976c0e7f-f1c9-4490-9c63-4b503617b402` |
| Vienna | vienna@rolliworks.com (also help@rolliworks.com seen earlier), role manager, id `8a92dfd0-933a-49ff-88b4-bb284d3d5a3c` |
| Test customer | Brian Poulson, sunflashx@gmail.com, conv `01a1edb8-3315-449d-a052-5a28e1aecd3a` |
| Multi-job test | Drew Hays, estimates #24932 + #24938 |
| Test reply token used | `27864080d72c3bec` (belongs to mike's client record) |
| RW band flow column | `jobs.flow` (STANDARD_FLOW / SPLIT_FLOW / BAND_ONLY_FLOW) |
| RS→RW intake endpoint | `rollisuite-intake` on RW project (caret-delimited, no auth) |

---

## Branch state

- RC has Test, staging, main branches
- Heavy divergence earlier resolved via Cursor merge (commit `1f26629`, kept Test versions on 4 conflict files)
- Lovable sometimes pushes direct to main
- Consider simplifying to main-only workflow for a 2-person team

---

## Tooling notes

- Claude (Anthropic) was unstable during this session — freezing, lag, app crashes. Server-side, not hardware.
- Lovable Publish deployed frontend without migration — verify migrations run on deploy going forward.
- Cursor handled git merge + diagnostics well; good for surgical work + verification.
- Clipboard sticking issues throughout (sometimes Claude dropped typed text, sometimes user clipboard stuck).

---

## Recommended next-session order

1. Clean up internal notes (remove leak risk) — FIRST
2. Restore reply token display
3. Verification pass on today's shipped items
4. Then resume feature work (estimate # lookup, band flow confirmation, etc.)
