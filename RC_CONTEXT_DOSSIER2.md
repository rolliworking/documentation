# RolliConnect (RC) — Context Dossier

> **Merged canonical version (2026-06-07).** Technical body derived by Lovable from the live RC
> codebase + read-only SELECT queries (project ref `ibwrjsmuvrqtokoqpogu`). Section 0 below is
> the human + cross-verified findings layer and is AUTHORITATIVE — it overrides any conflicting
> inference in the technical sections. If the technical body is ever regenerated, re-apply
> Section 0 on top.
>
> **Companion docs:** `NORTH-STAR.md` (suite vision — read first), `RS_CONTEXT_DOSSIER.md`,
> `RW_CONTEXT_DOSSIER.md`.

---

## 0. Verified Findings & Confirmed Decisions (read first — authoritative)

### 0.1 Thread fragmentation — root cause CONFIRMED, fix CONFIRMED
- **Root cause (Lovable code trace + cross-confirmed by live query):** exactly ONE code path,
  `supabase/functions/portal-intake-router/index.ts`, ever inserts a `conversations` row. It
  reuses the existing client but **always creates a new conversation** (step 3 of its flow),
  with no check for an existing open conversation. Every other path (portal composer, inbound
  email, team reply, photo upload) only ever appends. See §4 for the full trace.
- **Scale:** 63/500 clients (12.6%) have ≥2 conversations. Two independent measurements agree
  (Lovable all-conversations count 437/51/8/4; earlier active-only query 408/49/6/2). All
  multi-conversation cases originate from this one path.
- **INTENDED FIX (Michael, confirmed):** a new request from a client who already has a
  conversation should **merge into that existing conversation**, not fork. One client = one
  conversation. Two pieces of work:
  1. **Going-forward logic:** before step 3 insert, `portal-intake-router` must look up an
     existing active conversation for the `client_id` and route the new request into it;
     create only if none exists.
  2. **Backfill:** the 63 already-fractured clients' duplicate conversations merged into one
     (one-time data migration).
- `[FILL IN — Michael: distinguishability — when requests merge, keep each watch/job tracked
  individually WITHIN the thread (note: `conversation_jobs` currently has 0 rows, so it is NOT
  yet the mechanism), or flatten into one stream?]`
- `[FILL IN — Michael: backfill scope — merge the 63 existing fractured clients too, or
  going-forward only?]`

### 0.2 Portal process-flow — REFRAMED by evidence (was priority #1; now redirected)
- **Finding (Lovable, §5):** RC's portal Progress tab is **already fully data-driven**. It does
  NOT use a hardcoded linear flow. It calls RS's `rc-timeline-status` and renders whatever label
  strings RS returns — including unseen ones like `in_testing` / `uncased`, rendered verbatim
  with no error. The flow is open-ended on RC's side.
- **Implication:** the "wrong/confusing states" problem is NOT an RC display-logic bug. It
  originates in **RS** (the source of the status labels) and/or in how those labels read to a
  client. The portal-flow fix is therefore largely an **RS-side** concern (what status RS sends
  and whether the labels are client-appropriate), not an RC rebuild.
- `conversations.current_status` is **null across all 579 rows** — client-visible status lives
  entirely in RS, not mirrored in RC. The 20-entry `STATUS_OPTIONS` list (incl. "Uncased",
  "In Testing") exists in RC's CRM code but is unused in data.
- `[FILL IN — Michael: canonical source of truth for client-visible status — RS only, RC
  mirror, or hybrid? And the policy for client-facing wording of states like in-testing/uncased.]`

### 0.3 Priority recalibration (vs. NORTH-STAR.md §5)
- **Threading** is now the most rebuild-ready item in the suite: bounded scale, confirmed root
  cause, confirmed intended fix, surgical (one code path + a backfill). Strong first target.
- **Portal flow** turns out to be mostly an RS concern, not the broad RC bug assumed. Re-scope
  it as "review RS status labels for client-appropriateness," not an RC rebuild.

---

## 1. System overview

RolliConnect (RC) is the customer-facing communication layer of the Rolliworks ecosystem. It is hosted at `https://my.rolliworks.com` (also reachable via `crm.rolliworks.com`; Lovable preview at `*.lovable.app`). One React app serves two audiences from the same host:

- `/crm/*` — internal CRM for the Rolliworks team (Supabase Auth, gated by `team_users` + `user_roles`).
- `/client/*` — unauthenticated client portal, gated by a per-client unguessable URL token (`clients.token`) that is injected into every Supabase request as the `X-Client-Token` header.

### Stack

| Layer | Tech |
|---|---|
| Frontend | React 18.3, Vite 5.4, TypeScript 5.8, Tailwind 3.4, shadcn/ui (Radix), React Router 6.30, TanStack Query 5.83, React Hook Form + Zod |
| Backend | Supabase (`ibwrjsmuvrqtokoqpogu`) — Postgres + RLS, Edge Functions (Deno), Storage bucket `conversation-photos` |
| Email | Postmark (inbound webhook on `reply.rolliworks.com`) + Resend (outbound, via edge functions) |
| Hosting | Lovable Publish → `my-rolliworks-com.lovable.app` (canonical) + `my.rolliworks.com` custom domain |

### Relationship to RolliSuite (RS) and RolliWorking (RW)

RC has **its own dedicated Postgres database** (Supabase project `ibwrjsmuvrqtokoqpogu`). It does not read from or write to RS or RW tables directly. Integration happens at three boundaries:

| Direction | Mechanism | Where |
|---|---|---|
| RC → RS (read) | HTTPS POST to RS edge function `rc-timeline-status` with bearer `RS_TIMELINE_STATUS_KEY`; identified by `clients.rs_customer_id` | `supabase/functions/timeline-status/index.ts` |
| RC → RW (read) | HTTPS POST to RW edge functions `list-customer-approvals` and `list-customer-parts-approvals` with header `x-rc-service-key: RC_SERVICE_KEY`; identified by client email + `rs_customer_id` | `supabase/functions/list-customer-approvals/index.ts`, `supabase/functions/list-customer-parts-approvals/index.ts` |
| RS → RC (read) | HTTPS POST to RC edge function `get-rc-portal-link` with header `x-rc-service-key: RC_SERVICE_KEY`; returns the portal URL for an email or `rs_customer_id` | `supabase/functions/get-rc-portal-link/index.ts` |

**Shared identifier:** `clients.rs_customer_id uuid` (nullable) ↔ RS customer UUID. RC stores it locally; it is set at intake (`portal-intake-router` payload) or backfilled by the one-off `backfill-rs-customer-ids` function. There are no foreign keys to or from external systems — the link is by value, not by FK.

There is no webhook **from** RS or RW into RC; all cross-system reads are pull-based, on-demand from the user's session (timeline + approvals are fetched live when the client opens the portal).

[FILL IN — RC's business identity, target customer, and intended client journey beyond "watch service requests".]

[FILL IN — Who the operators of RC are day-to-day (number of staff, shifts), and the volume targets the system is sized for.]

---

## 2. Full schema

16 base tables in `public` (15 named in the brief plus `conversation_views`, which exists in the database and is in active use). All FKs were enumerated via `information_schema.table_constraints` and returned **zero rows** — i.e. no declared foreign-key constraints exist in the database. Logical references are documented in the "Logical FK" column below; they are enforced only in application code, not at the DB level.

```sql
select tc.table_name, kcu.column_name, ccu.table_name as ftable, ccu.column_name as fcol
from information_schema.table_constraints tc
join information_schema.key_column_usage kcu on tc.constraint_name=kcu.constraint_name
join information_schema.constraint_column_usage ccu on tc.constraint_name=ccu.constraint_name
where tc.constraint_type='FOREIGN KEY' and tc.table_schema='public';
-- → 0 rows
```

### `clients`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `token` | text | NO | — | (portal URL token, unique-by-convention) |
| `email` | text | NO | — | |
| `phone` | text | YES | — | |
| `name` | text | NO | — | |
| `status` | text | NO | `'prospect'` | |
| `first_seen_at` | timestamptz | NO | `now()` | |
| `last_activity_at` | timestamptz | NO | `now()` | |
| `archived` | bool | NO | `false` | |
| `created_at` | timestamptz | NO | `now()` | |
| `updated_at` | timestamptz | NO | `now()` | |
| `rs_customer_id` | uuid | YES | — | → RS `customers.id` (by value, no FK) |
| `reply_token` | text | NO | — | (legacy 16-hex token from intake) |

### `conversations`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `client_id` | uuid | NO | — | → `clients.id` |
| `service_type` | text | NO | — | free text, populated from intake form |
| `initial_description` | text | YES | — | |
| `intake_lead_id` | uuid | YES | — | (external lead reference) |
| `source_entity` | text | NO | `'rolliworks'` | currently always `rolliworks` |
| `state` | text | NO | `'awaiting_team'` | inbox routing — see §6 |
| `completed_at` | timestamptz | YES | — | unused in current code paths |
| `archived_at` | timestamptz | YES | — | |
| `last_activity_at` | timestamptz | NO | `now()` | |
| `last_nudged_at` | timestamptz | YES | — | |
| `created_at` | timestamptz | NO | `now()` | |
| `updated_at` | timestamptz | NO | `now()` | |
| `last_message_at` | timestamptz | YES | — | maintained by trigger `update_conversation_last_message_at` |
| `current_status` | text | YES | — | **declared but unused** — see §5/§6 |
| `status_updated_at` | timestamptz | YES | — | unused |
| `status_updated_by` | uuid | YES | — | unused |
| `category` | text | NO | `'conversations'` | inbox tab; see §6 |

### `messages`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `conversation_id` | uuid | NO | — | → `conversations.id` |
| `author_type` | text | NO | — | `'client' | 'team' | 'system'` |
| `author_id` | uuid | YES | — | → `team_users.id` when team |
| `author_display_name` | text | NO | `'Rolliworks'` | |
| `body` | text | NO | — | |
| `is_auto_generated` | bool | NO | `false` | |
| `created_at` | timestamptz | NO | `now()` | |
| `inbound_sender_mismatch` | bool | NO | `false` | true when Postmark From != client email |
| `inbound_postmark_message_id` | text | YES | — | dedupe key |
| `template_id` | uuid | YES | — | → `saved_replies.id` |
| `is_internal` | bool | NO | `false` | hidden from portal RLS |

### `conversation_jobs`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `conversation_id` | uuid | NO | — | → `conversations.id` |
| `job_id` | uuid | NO | — | → RS estimate/job id (by value) |
| `created_at` | timestamptz | NO | `now()` | |

### `conversation_labels`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `conversation_id` | uuid | NO | — | → `conversations.id` |
| `label` | text | NO | — | |
| `set_by` | uuid | YES | — | → `team_users.id` |
| `set_at` | timestamptz | NO | `now()` | |

### `conversation_read_states`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `conversation_id` | uuid | NO | — | → `conversations.id` (effectively PK, used as `onConflict`) |
| `last_read_at` | timestamptz | YES | — | |
| `marked_unread_at` | timestamptz | YES | — | |
| `is_read` | bool | NO | `false` | shared across the team (no per-user row) |

### `conversation_views`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `conversation_id` | uuid | YES | — | → `conversations.id` (null on landing page) |
| `viewer_type` | text | NO | — | `'client' | 'team'` |
| `viewer_id` | uuid | YES | — | |
| `ip_address` | text | YES | — | |
| `user_agent` | text | YES | — | |
| `viewed_at` | timestamptz | NO | `now()` | |

### `conversation_photos`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `conversation_id` | uuid | NO | — | → `conversations.id` |
| `uploaded_by_type` | text | NO | — | `'client' | 'team'` |
| `uploaded_by_team_user_id` | uuid | YES | — | → `team_users.id` |
| `storage_path` | text | NO | — | path in bucket `conversation-photos` |
| `filename` | text | NO | — | |
| `mime_type` | text | YES | — | |
| `size_bytes` | int | YES | — | |
| `message_id` | uuid | YES | — | → `messages.id` (null = standalone upload) |
| `created_at` | timestamptz | NO | `now()` | |

### `email_templates`
| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` |
| `template_key` | text | NO | — |
| `subject` | text | NO | — |
| `body` | text | NO | — |
| `updated_at` | timestamptz | NO | `now()` |
| `updated_by` | text | YES | — |
| `last_sent_at` | timestamptz | YES | — |

### `saved_replies`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `user_id` | uuid | NO | — | → `auth.users.id` |
| `label` | text | NO | — | |
| `body` | text | NO | — | |
| `sort_order` | int | NO | `0` | |
| `created_at` | timestamptz | NO | `now()` | |
| `updated_at` | timestamptz | NO | `now()` | |
| `attachment_storage_path` | text | YES | — | |
| `attachment_filename` | text | YES | — | |
| `attachment_mime_type` | text | YES | — | |
| `last_used_at` | timestamptz | YES | — | |
| `is_trigger` | bool | NO | `false` | when true → routes via `route_conversation_on_message` |
| `move_to_category` | text | YES | — | target inbox category for trigger replies |

### `nudge_queue`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `conversation_id` | uuid | NO | — | → `conversations.id` |
| `scheduled_for` | timestamptz | NO | — | |
| `sent_at` | timestamptz | YES | — | |
| `cancelled_at` | timestamptz | YES | — | |
| `created_at` | timestamptz | NO | `now()` | |

### `unmatched_inbound`
| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` |
| `received_at` | timestamptz | NO | `now()` |
| `from_email` / `from_name` / `to_address` / `subject` | text | YES | — |
| `body_preview` | text | YES | — |
| `raw_payload` | jsonb | YES | — |
| `reason` | text | YES | — |
| `reviewed_at` | timestamptz | YES | — |

### `outbound_attachment_hashes`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `hash_sha256` | text | NO | — | sha256 of attachment bytes |
| `conversation_id` | uuid | YES | — | → `conversations.id` |
| `message_id` | uuid | YES | — | → `messages.id` |
| `filename` | text | YES | — | |
| `bytes` | int | YES | — | |
| `created_at` | timestamptz | NO | `now()` | |

### `team_users`
| Column | Type | Null | Default |
|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` |
| `email` | text | NO | — |
| `first_name` | text | NO | — |
| `display_name` | text | NO | — |
| `role` | text | NO | — |
| `entity_access` | text[] | NO | `ARRAY['rolliworks']` |
| `created_at` | timestamptz | NO | `now()` |

### `user_roles`
| Column | Type | Null | Default | Logical FK |
|---|---|---|---|---|
| `id` | uuid | NO | `gen_random_uuid()` | — |
| `user_id` | uuid | NO | — | → `auth.users.id` |
| `role` | `app_role` enum (`admin | moderator | user`) | NO | — | |
| `created_at` | timestamptz | NO | `now()` | |

### `app_settings`
Singleton row (`id = 1`).
| Column | Type | Null | Default |
|---|---|---|---|
| `id` | int | NO | `1` |
| `rc_email_mode` | text | NO | `'test'` |
| `email_welcome_enabled` | bool | NO | `false` |
| `email_reply_notification_enabled` | bool | NO | `true` |
| `email_nudge_enabled` | bool | NO | `false` |
| `updated_at` | timestamptz | NO | `now()` |
| `updated_by` | uuid | YES | — |

### Database functions / triggers

| Function | Purpose |
|---|---|
| `current_client_token()` | Reads the portal `X-Client-Token` from `request.headers`; used by RLS policies on `clients`, `conversations`, `messages`, `conversation_photos`, `conversation_views`, `nudge_queue`. |
| `has_role(uuid, app_role)` | SECURITY DEFINER role check used in admin RLS. |
| `handle_new_user_admin_role()` | On new `auth.users` row, if email matches a `team_users` admin, insert `user_roles` admin. |
| `update_conversation_last_message_at()` | Trigger on `messages` insert → bumps `conversations.last_message_at`. |
| `route_conversation_on_message()` | Trigger on `messages` insert → routes the conversation between `category` values, un-archives on client reply, archives on trigger saved replies. |
| `get_inbox_unread_counts()` | Aggregates unread counts (client messages + manual unread flags). |
| `update_updated_at_column()` | Generic `updated_at` bump. |

`information_schema.triggers` reports no triggers, but the trigger functions above are referenced by the application; the `update_conversation_last_message_at` and `route_conversation_on_message` functions are wired via migration files. [FILL IN — confirm whether these triggers are currently attached in the live DB; the `information_schema.triggers` listing came back empty in tooling.]

---

## 3. Module / route map

Routing entry point: `src/App.tsx`. On hosts containing `my.` or any other host, the `CombinedRoutes` tree is mounted; on `crm.*` the CRM-only tree is mounted. Path layout below applies to `my.rolliworks.com`.

### Client portal (unauthenticated, token-gated)

| Path | Component | Role |
|---|---|---|
| `/client/:token` | `pages/portal/ClientLanding.tsx` | Resolves client by token; lists their non-archived conversations. If exactly one exists, auto-redirects to it. Logs a `conversation_views` row with `conversation_id = null`. |
| `/client/:token/c/:conversation_id` | `pages/portal/ClientConversation.tsx` | Main portal screen. Tabbed UI: **Messages**, **Progress**, **Photos** (coming-soon stub), **Documents**. Composer posts to `messages` with `author_type='client'` and updates conversation state. |
| `/client/:token/inspection/:approval_id` | `pages/portal/InspectionViewPage.tsx` | Read-only render of an RW inspection approval (item details). |
| `/client/*` (fallback) | `pages/portal/InvalidLink.tsx` | Error landing. |

Supporting components: `components/portal/DocumentsList.tsx` (inspection + parts approvals), `components/portal/InspectionFindingsGrid.tsx`, `components/portal/TCAcceptancePanel.tsx`, `components/portal/ComingSoonTab.tsx`, `components/PortalHeader.tsx`, `components/PhotoUpload.tsx`, `components/InlinePhotos.tsx`.

Portal Supabase client (`src/integrations/supabase/portal-client.ts`) injects the URL token as the `X-Client-Token` header; RLS policies (see `clients`, `conversations`, `messages`, `conversation_photos`) read it via `current_client_token()`.

### CRM (Supabase Auth + `team_users`)

| Path | Component | Role |
|---|---|---|
| `/login` | `pages/crm/Login.tsx` | Supabase Auth email/password login. |
| `/` | `RootRedirect` | Redirects to `/crm/inbox` if logged in else `/login`. |
| `/crm/inbox` | `pages/crm/CrmInbox.tsx` | Inbox grid driven by `useInboxRows`; tabs filter by `conversations.category`. |
| `/crm/c/:conversation_id` | `pages/crm/CrmConversation.tsx` | Team conversation view; uses `components/crm/ConversationPanel.tsx`. |
| `/crm/clients/:client_id` | `pages/crm/ClientProfile.tsx` | Per-client overview + conversation list. |
| `/crm/admin/team` | `pages/crm/AdminTeam.tsx` | Admin-only `team_users` management. |
| `/crm/settings` | `pages/crm/Settings.tsx` | Admin email-mode + template editor (`EmailTemplatesEditor.tsx`). |
| `/crm/search` | `pages/crm/CrmSearchResults.tsx` | Cross-conversation search. |
| Legacy `/c/:id`, `/clients/:id`, `/admin/team`, `/settings` | redirects to `/crm/*`. |

CRM auth flow: `hooks/useAuth.tsx` → Supabase session → match `team_users` by lowercased email → `user_roles` for `admin` checks.

### Edge functions (`supabase/functions/`)

| Function | verify_jwt | Role |
|---|---|---|
| `portal-intake-router` | `false` | **Sole creator of `conversations` rows.** Receives intake form, creates-or-reuses `clients`, always inserts a new `conversations`, posts system intake message, schedules welcome email. |
| `inbound-email-handler` | `false` | Postmark webhook. Resolves HMAC token from recipient → existing conversation, inserts a client message, marks conversation `awaiting_team`. Never creates conversations. |
| `portal-send-welcome` | default | Sends welcome email after intake. |
| `portal-send-reply-notification` | default | Notifies client when team replies (Reply-To uses HMAC inbound token). |
| `portal-send-nudge` | default | Sends one nudge email for a conversation. |
| `portal-nudge-runner` | default | Cron (~15 min) processing `nudge_queue`. |
| `timeline-status` | default | RC → RS proxy for live job timeline. |
| `list-customer-approvals` | default | RC → RW proxy for inspection approvals. |
| `list-customer-parts-approvals` | default | RC → RW proxy for parts approvals. |
| `get-rc-portal-link` | default | RS → RC: returns portal URL by email or `rs_customer_id`. |
| `get-conversation-reply-address` | `false` (default) | Returns the per-conversation `*@reply.rolliworks.com` address. |
| `backfill-rs-customer-ids` | `false` | One-off backfill. |

---

## 4. Conversation-creation logic — root of thread fragmentation

Repository-wide search for inserts into `conversations`:

```bash
rg -n 'from\("conversations"\)\.insert' src/ supabase/functions/
# → supabase/functions/portal-intake-router/index.ts:93
# (no other matches anywhere in the codebase)
```

**There is exactly one code path that creates a `conversations` row: `supabase/functions/portal-intake-router/index.ts`.** No CRM screen, edge function, or portal page creates conversations. Inbound email, portal composer, photo upload, and timeline/approval reads all assume the conversation already exists.

### `portal-intake-router` flow (the only constructor)

Input: `{ email, name, phone?, service_type, initial_description?, intake_lead_id?, rs_customer_id? }`. Returns `{ client_token, conversation_id, is_new_client }`.

Pseudocode (verbatim shape from `supabase/functions/portal-intake-router/index.ts`):

```text
1. Lower-case + trim email.
2. SELECT clients WHERE email = $email.
   - if exists: reuse client_id and client.token; update last_activity_at;
                if rs_customer_id was missing, backfill it.
   - else:     INSERT clients with a freshly generated token + reply_token; is_new_client = true.
3. INSERT conversations { client_id, service_type, initial_description, intake_lead_id,
                          state='awaiting_team', category='requests' }.   ← ALWAYS NEW
4. INSERT messages { conversation_id, author_type='system', author_display_name='Rolliworks',
                     body=<intake summary>, is_auto_generated=true }.
5. UPSERT conversation_read_states { conversation_id, is_read=false, marked_unread_at=now }
   on conflict (conversation_id).
6. Fire portal-send-welcome (non-blocking) with { conversation_id, is_new_client }.
```

### When a NEW conversation is created vs appended

| Trigger | Path | Creates new `conversations`? | Notes |
|---|---|---|---|
| Intake-form submission (any time, even for an existing client) | `portal-intake-router` | **YES — unconditionally.** | No lookup of existing open/recent conversation for the same `client_id`. The client is reused; the conversation is not. |
| Client reply via portal composer | `pages/portal/ClientConversation.tsx` → `messages.insert` | No — appends to current `conversation_id` from URL. |
| Client inbound email reply | `supabase/functions/inbound-email-handler/index.ts` | No — HMAC token in `To:` resolves to an existing conversation; on miss, the email is dropped into `unmatched_inbound`. Never creates. |
| Team reply (CRM) | `components/crm/ConversationPanel.tsx` | No — appends to the open conversation. |
| Photo upload (portal or CRM) | `components/PhotoUpload.tsx` | No — attaches to existing conversation. |

### Implication (the fragmentation root cause)

Every intake-form submission produces a brand-new `conversations` row, even when the submitting client already has open conversations. There is no dedupe window, no "open conversation per service_type" check, no merging. Aggregate evidence:

```sql
with x as (select client_id, count(*) c from conversations group by client_id)
select c as convs_per_client, count(*) as clients from x group by 1 order by 1;
--  convs_per_client | clients
--  ----------------+---------
--                1 |     437
--                2 |      51
--                3 |       8
--                4 |       4
```

```sql
with x as (select client_id, count(*) c from conversations group by client_id)
select count(*) filter (where c>1) as multi_conv_clients, count(*) as total_clients from x;
--  multi_conv_clients | total_clients
--  -------------------+---------------
--                  63 |           500
```

63 / 500 = **12.6 %** of clients have ≥2 conversations. All multi-conversation cases originate from this one code path; the inbound + portal-composer paths can only ever extend an existing thread. [FILL IN — desired business behaviour: should a second intake submission within N days for the same client merge into the open conversation, or always fork?]

---

## 5. Client portal process flow (Progress tab)

The Progress tab is rendered by `ProgressView` inside `src/pages/portal/ClientConversation.tsx` (lines 441–566). It does **not** read any local status field on `conversations`. Instead, it invokes the `timeline-status` edge function, which proxies to RolliSuite (`rc-timeline-status`) keyed by `clients.rs_customer_id`.

### Data shape returned to the portal

```ts
{
  current_step_label: string | null,
  next_step_label:    string | null,
  secondary_current_label: string | null,   // only when status.version === 2
  updated_at: string | null,
  version: number | null,
}
```

The UI is **fully data-driven**: it renders whatever label strings RS returns. There is no hardcoded sequence in RC; RC does not know the list of possible steps, the order, or which step comes next. The only step-specific UI logic in RC is a cosmetic "(if needed)" caption rendered when the label string equals `"Parts Pending Approval"` — see `ClientConversation.tsx:526` and `:547`.

### Fallback / unknown-state handling

| Condition | UI shown |
|---|---|
| `client.rs_customer_id` is null → `timeline-status` returns `{ ok: true, status: null, reason: 'no_rs_customer' }` | "Pending update" heading |
| RS returns non-ok | "Pending update" |
| RS returns ok but `current_step_label` is null and `next_step_label` is null | "Pending update" → "All steps complete." caption appears if `current_step_label` is set but `next_step_label` is null |
| Any label string RC has never seen (e.g. `in_testing`, `uncased`) | Rendered verbatim as the pill text — no special handling, no error. The flow is open-ended. |

### Static status list in RC (separate from the Progress tab)

`src/lib/statusOptions.ts` exports a 20-entry `STATUS_OPTIONS` constant:

```ts
"Initial Request", "Estimate Created", "Estimate Sent", "Shipping Label Requested",
"Shipping Label Sent", "Package Received", "Intake Photos", "Client Asset in Inventory",
"Intake Labels Printed", "Inspection Photos", "Inspection Completed", "In Queue",
"Uncased", "In Progress", "In Testing", "Parts Pending Approval", "Parts on Order",
"Work Complete", "Sales Order", "Exit Photos"
```

This list is used in CRM UI only (`components/crm/ConversationPanel.tsx` references `current_status` / `status_updated_at`). It is **not** read by the portal Progress tab, and the column it would populate (`conversations.current_status`) is empty across the dataset:

```sql
select coalesce(current_status,'(null)'), count(*) from conversations group by 1 order by 2 desc;
--  coalesce | count
--  ---------+-------
--  (null)   |   579
```

So in production today, RC stores no local status, and the portal flow is entirely dependent on the RS `rc-timeline-status` response. [FILL IN — intended canonical source of truth for client-visible status: RS only, RC mirror, or hybrid? And the policy for displaying RS steps RC has never been told about.]

---

## 6. Inbox routing (`state` + `category`)

`conversations.state` controls who is "waiting" (drives reminder logic and portal labels). `conversations.category` controls which CRM tab the row lives in. They are independent.

- `state` transitions: set to `'awaiting_team'` on intake and on every client reply (composer + inbound email). Set to `'awaiting_client'` when team replies (in `ConversationPanel.tsx`). Set to `'archived'` by trigger saved replies. Un-archived back to `'awaiting_team'` by the routing trigger on any client reply.
- `category` transitions: defaults to `'requests'` at intake; trigger saved replies (`saved_replies.is_trigger=true`) move it to `saved_replies.move_to_category`; CRM bulk actions in `CrmInbox.tsx` set `'quoted'` or `'answered'`; any client reply forces it back to `'conversations'`.

---

## 7. Behavioural data (aggregate)

All counts below were produced from the queries shown immediately above them, run read-only against the live RC database.

### Table row counts

```sql
select table_name,
       (xpath('/row/c/text()',
              query_to_xml(format('select count(*) as c from public.%I', table_name),
                           false, true, '')))[1]::text::int as n
from information_schema.tables
where table_schema='public' and table_type='BASE TABLE'
order by table_name;
```

| Table | Rows |
|---|---:|
| `app_settings` | 1 |
| `clients` | 500 |
| `conversation_jobs` | 0 |
| `conversation_labels` | 39 |
| `conversation_photos` | 624 |
| `conversation_read_states` | 579 |
| `conversation_views` | 5365 |
| `conversations` | 579 |
| `email_templates` | 4 |
| `messages` | 1952 |
| `nudge_queue` | 805 |
| `outbound_attachment_hashes` | 2 |
| `saved_replies` | 32 |
| `team_users` | 4 |
| `unmatched_inbound` | 6 |
| `user_roles` | 2 |

### `conversations.state`

```sql
select state, count(*) from conversations group by state order by 2 desc;
```
| state | count |
|---|---:|
| awaiting_client | 332 |
| awaiting_team | 201 |
| archived | 46 |

### `conversations.current_status`

```sql
select coalesce(current_status,'(null)'), count(*) from conversations group by 1 order by 2 desc;
```
| current_status | count |
|---|---:|
| (null) | 579 |

Field is declared but unused in production data.

### `conversations.category`

```sql
select category, count(*) from conversations group by 1 order by 2 desc;
```
| category | count |
|---|---:|
| answered | 314 |
| requests | 97 |
| quoted | 70 |
| conversations | 58 |
| archived | 40 |

### `conversations.service_type`

```sql
select service_type, count(*) from conversations group by 1 order by 2 desc limit 30;
```
| service_type | count |
|---|---:|
| Bracelet Repair | 244 |
| Rolex Movement Service | 89 |
| Other | 58 |
| Movement + Case + Bracelet Work | 44 |
| Movement + Bracelet Work | 41 |
| Service Inquiry | 35 |
| Rolex Case Repair | 28 |
| Bracelet + Case Work | 21 |
| Rolex Case or Bracelet Polish | 11 |
| Service request | 6 |
| watch | 1 |
| Diagnostic | 1 |

Free-text field; the two singletons (`watch`, `Diagnostic`) and the lowercase `Service request` indicate the intake form has historically allowed/used non-canonical values. [FILL IN — whether `service_type` should be normalized to an enum.]

### `conversations.source_entity`

```sql
select source_entity, count(*) from conversations group by 1 order by 2 desc;
```
| source_entity | count |
|---|---:|
| rolliworks | 579 |

Single value across all rows; the column is currently effectively unused.

### Conversations created per month (last 12 months)

```sql
select date_trunc('month', created_at)::date as m, count(*)
from conversations
where created_at > now() - interval '12 months'
group by 1 order by 1;
```
| month | count |
|---|---:|
| 2026-04-01 | 20 |
| 2026-05-01 | 439 |
| 2026-06-01 | 120 |

Only three months of history exist in the dataset — RC is a young system or recently migrated. [FILL IN — was data backfilled, or is this the true production lifetime?]

### Message volume by author

```sql
select author_type, count(*) from messages group by 1 order by 2 desc;
```
| author_type | count |
|---|---:|
| team | 831 |
| system | 579 |
| client | 542 |

`system` count exactly equals `conversations` count (579) — every conversation has the one auto-generated intake summary, as expected from `portal-intake-router`. Team:client ratio is ≈1.53 outbound team messages per inbound client message.

### `conversation_jobs` linkage

```sql
select count(distinct conversation_id) as conversations_with_job,
       count(*) as job_links
from conversation_jobs;
```
| conversations_with_job | job_links |
|---|---:|
| 0 | 0 |

No conversation is currently linked to an RS job/estimate in the RC database. [FILL IN — confirm whether `conversation_jobs` is intended to be populated by RC, by RS (write-back), or is deprecated.]

### `unmatched_inbound`

```sql
select coalesce(reason,'(null)'), count(*) from unmatched_inbound group by 1 order by 2 desc;
```
| reason | count |
|---|---:|
| token_no_match | 6 |

Six inbound emails total have ever landed in the dead-letter table, all with `reason = 'token_no_match'` (recipient included a token-formatted address but the token did not resolve to a live conversation). No `no_token_in_recipient` or `secret_missing` rows.

### Conversations per client

```sql
with x as (select client_id, count(*) c from conversations group by client_id)
select c as convs_per_client, count(*) as clients from x group by 1 order by 1;
```
| convs_per_client | clients |
|---|---:|
| 1 | 437 |
| 2 | 51 |
| 3 | 8 |
| 4 | 4 |

```sql
with x as (select client_id, count(*) c from conversations group by client_id)
select count(*) filter (where c>1) as multi_conv_clients, count(*) as total_clients from x;
```
| multi_conv_clients | total_clients |
|---|---:|
| 63 | 500 |

---

## 8. Open questions / `[FILL IN]` index

1. RC's business identity, target customer, and intended client journey beyond "watch service requests". (§1)
2. Who the operators of RC are day-to-day (number of staff, shifts), and the volume targets the system is sized for. (§1)
3. Confirm whether the `update_conversation_last_message_at` and `route_conversation_on_message` triggers are currently attached in the live DB (`information_schema.triggers` returned empty in tooling). (§2)
4. Desired business behaviour for repeat intake submissions: merge into open conversation within a window vs. always fork. (§4)
5. Intended canonical source of truth for client-visible status: RS only, RC mirror, or hybrid — and policy for displaying RS steps RC has never been told about. (§5)
6. Whether `service_type` should be normalized to an enum. (§7)
7. Whether the only-three-months-of-data window reflects a recent migration / backfill or the true production lifetime. (§7)
8. Whether `conversation_jobs` is intended to be populated by RC, by RS write-back, or is deprecated (currently 0 rows). (§7)

---

End of dossier.
---

## 9. Reconciliation note (Section 0 answers to §8 open questions)

- §8 Q4 (repeat intake: merge vs fork) → **ANSWERED in §0.1: merge into existing conversation.**
  Open sub-decisions: distinguishability + backfill scope.
- §8 Q5 (canonical status source) → **partially answered in §0.2:** status currently lives in
  RS only (RC `current_status` is null). Final canonical-source policy still `[FILL IN]`.
- §8 Q8 (`conversation_jobs` 0 rows) → relevant to §0.1 distinguishability decision: if merged
  threads must track jobs individually, `conversation_jobs` would need to be populated (it is
  not today).
- Remaining §8 items (business identity, operators, trigger attachment, service_type enum, data
  window) remain open for Michael / Qwen.
