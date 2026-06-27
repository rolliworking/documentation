# Q-003 — Map the RolliConnect inbox + portal

**Status:** in-progress
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-26
**Skill:** `mapping-legacy-workflows`

---

## 1. RolliConnect (RC) — architectural overview

**Purpose:** A customer-facing portal + team CRM for managing client conversations. RC's domain is **customer communication** — not job management (that's RS) and not shop floor workflow (that's RW).

**Three user types:**
1. **Clients** — anonymous portal users (token-based, no login)
2. **Team responders** — authenticated CRM users (login + team role)
3. **Admins** — team users with `admin` role (can manage team_users and app settings)

**Infrastructure:** RC is a standalone Supabase project with its own database. Data is **not shared** with RS or RW. Integration happens via:
- `rs_customer_id` field on the `clients` table (links RC client to RS customer)
- Edge function calls (portal link generation, timeline status fetch)

---

## 2. Inbox architecture (CRM)

### Core tables

#### `conversations` — the central unit

```
CREATE TABLE public.conversations (
  id uuid PK,
  client_id uuid REFERENCES clients(id) ON DELETE CASCADE,
  service_type text NOT NULL,          -- e.g., "jewelry repair", "watch service"
  initial_description text,
  intake_lead_id uuid,
  source_entity text DEFAULT 'rolliworks',
  state text DEFAULT 'awaiting_team' CHECK (
    state IN ('awaiting_team', 'awaiting_client', 'converted', 'stale', 'archived')
  ),
  flag text CHECK (flag IN ('red', 'green', 'yellow') OR flag IS NULL),
  flag_set_by uuid REFERENCES team_users(id),
  flag_set_at timestamptz,
  completed_at timestamptz,
  archived_at timestamptz,
  last_activity_at timestamptz NOT NULL,
  last_nudged_at timestamptz,
  created_at/updated_at timestamptz
);
```

**State machine (conversations):**
```
awaiting_team ←→ awaiting_client
        ↓            ↓
     converted    stale
        ↓
      archived
```

- **`awaiting_team`** — Client has sent something, team needs to respond
- **`awaiting_client`** — Team has responded, waiting for client reply
- **`converted`** — Service performed (mapped from RS job status `converted`)
- **`stale`** — Client has been silent for too long
- **`archived`** — Conversation is done (manual or auto-archival)

#### `messages` — the conversation content

```
CREATE TABLE public.messages (
  id uuid PK,
  conversation_id uuid REFERENCES conversations(id) ON DELETE CASCADE,
  author_type text CHECK (
    author_type IN ('client', 'team', 'system')
  ),
  author_id uuid,            -- user ID (team) or null (client/system)
  author_display_name text NOT NULL DEFAULT 'Rolliworks',
  body text NOT NULL,
  is_auto_generated boolean DEFAULT false,
  inbound_sender_mismatch boolean DEFAULT false,  -- email sender ≠ client email
  inbound_postmark_message_id text,               -- Postmark ID for dedupe
  created_at timestamptz
);
```

**Message author types:**
- **`client`** — Written by client (via email reply or portal)
- **`team`** — Written by team responder (via CRM UI)
- **`system`** — Auto-generated (initial request, welcome messages, etc.)

#### `conversation_jobs` — link to RS jobs

```
CREATE TABLE public.conversation_jobs (
  id uuid PK,
  conversation_id uuid REFERENCES conversations(id) ON DELETE CASCADE,
  job_id uuid NOT NULL,
  created_at timestamptz,
  UNIQUE (conversation_id, job_id)
);
```

This links conversations to RS jobs via `job_id`. Per the RS dossier (Q-001), RS jobs link to RS customers, and RC `clients` have `rs_customer_id`. So the full chain is:
`RC conversation → RC client → RS customer → RS job`

#### `conversation_read_states` — team read tracking

```
CREATE TABLE public.conversation_read_states (
  id uuid PK,
  conversation_id uuid REFERENCES conversations(id),
  team_user_id uuid REFERENCES team_users(id),
  is_read boolean DEFAULT false,
  last_read_at timestamptz,
  marked_unread_at timestamptz,
  UNIQUE (conversation_id, team_user_id)
);
```

**Key insight:** This table tracks **per-team-member** read state for each conversation. When a team member marks a conversation as read (by viewing it), `last_read_at` is set. When marked unread (manually), `marked_unread_at` is set. The `get_inbox_unread_counts` RPC combines manual and automatic unread tracking.

#### `conversation_views` — audit of who viewed what

```
CREATE TABLE public.conversation_views (
  id uuid PK,
  conversation_id uuid REFERENCES conversations(id),
  viewer_type text CHECK (viewer_type IN ('client', 'team')),
  viewer_id uuid,
  ip_address text,
  user_agent text,
  viewed_at timestamptz DEFAULT now()
);
```

#### `conversation_labels` / `conversation_photos` / `saved_replies`

- **`conversation_labels`** — Color-coded labels per conversation (yellow/red/green)
- **`conversation_photos`** — Photos uploaded by either client or team, stored in Supabase Storage bucket `conversation-photos`
- **`saved_replies`** — Team responders can save templates for quick insertion (with `sort_order` and `last_used_at`)
- **`outbound_attachment_hashes`** — Deduplication tracker for outbound attachment hashes
- **`unmatched_inbound`** — Catch-all for inbound emails that couldn't be routed
- **`email_templates`** — Template storage: `reply_notification`, `nudge`, `welcome_new_client`, `welcome_returning_client`

#### `app_settings` — singleton configuration

```
CREATE TABLE public.app_settings (
  id integer PK DEFAULT 1,
  rc_email_mode text DEFAULT 'test' CHECK (rc_email_mode IN ('test', 'live')),
  email_welcome_enabled boolean,
  updated_at timestamptz,
  updated_by uuid REFERENCES team_users(id),
  CONSTRAINT app_settings_singleton CHECK (id = 1)
);
```

#### `clients` table

```
CREATE TABLE public.clients (
  id uuid PK,
  token text UNIQUE NOT NULL,              -- 64-char random token (portal URL key)
  email text UNIQUE NOT NULL,              -- ← THE DUPLICATE CLIENT ISSUE
  phone text,
  name text NOT NULL,
  status text DEFAULT 'prospect' CHECK (status IN ('prospect', 'client')),
  first_seen_at timestamptz,
  last_activity_at timestamptz,
  archived boolean DEFAULT false,
  rs_customer_id uuid REFERENCES RS.customers(id),  -- links to RS customer
  reply_token text UNIQUE NOT NULL,        -- 16 hex reply token
  created_at/updated_at timestamptz
);
```

#### `team_users` table

```
CREATE TABLE public.team_users (
  id uuid PK,
  email text UNIQUE NOT NULL,
  first_name text NOT NULL,
  display_name text NOT NULL,
  role text CHECK (role IN ('admin', 'responder')),
  entity_access text[] DEFAULT ARRAY['rolliworks'],
  created_at timestamptz
);
```

### CRM UI Components

| Page/Component | File path | Purpose |
|---|---|---|
| **CrmInbox** | `pages/crm/CrmInbox.tsx` | Split-pane inbox (list + panel) |
| **ConversationPanel** | `components/crm/ConversationPanel` | Message threading + reply + photo gallery |
| **CrmSearchResults** | `pages/crm/CrmSearchResults.tsx` | Search results for clients |
| **ClientProfile** | `pages/crm/ClientProfile.tsx` | Client detail page with active conversations |
| **InboxSearchContext** | `contexts/InboxSearchContext.tsx` | Search state context |
| **CrmHeaderSearch** | `components/CrmHeaderSearch.tsx` | Search bar in CRM header |
| **SavedRepliesCarousel** | `components/SavedRepliesCarousel.tsx` | Saved replies quick-access |
| **InlinePhotos** | `components/InlinePhotos.tsx` | Photo display in messages |
| **PhotoUpload** | `components/PhotoUpload.tsx` | Photo upload (for team) |
| **PhotosLightbox** | `components/PhotoLightbox.tsx` | Photo zoom viewer |
| **EmailModeBadge** | `components/EmailModeBadge.tsx` | Shows test/live email mode |
| **EmailTemplatesEditor** | `components/EmailTemplatesEditor.tsx` | Edit email templates |
| **DeleteSavedReplyDialog** | `components/DeleteSavedReplyDialog.tsx` | Delete saved reply |

### CRM inbox query logic (useInboxRows hook)

The inbox uses a 4-way parallel query:
1. **conversations** — Last 500, ordered by `last_message_at` desc
2. **clients** — Join to get name/email for each conversation's client
3. **messages** — Last message per conversation (for preview)
4. **conversation_labels** — Labels per conversation
5. **get_inbox_unread_counts RPC** — Per-team-user unread counts per conversation

### CRM inbox tabs

Five category tabs (stored as `category` column on conversations):
1. **Conversations** (`category = 'conversations'`)
2. **Requests** (`category = 'requests'`)
3. **Quoted** (`category = 'quoted'`)
4. **Answered** (`category = 'answered'`)
5. **Archived** (`category = 'archived'`)

### CRM inbox actions

- **Toggle label** (yellow/red/green) — adds/removes conversation labels
- **Archive** — sets `category = 'archived'`, `state = 'archived'`, `archived_at = now`
- **Move to Quoted** — sets `category = 'quoted'`
- **Move to Answered** — sets `category = 'answered'`
- **Delete key** — shortcut: moves selected to Answered
- **Search** — filters by client name or email

---

## 3. Client portal architecture

### Portal URL pattern

```
https://my.rolliworks.com/client/:token          — Client landing (list of active requests)
https://my.rolliworks.com/client/:token/c/:convId — Specific conversation
```

### Portal authentication (token-based, no login)

Portal clients authenticate via **URL path token injection**:

```typescript
// portal-client.ts
// Creates an anonymous Supabase client that injects X-Client-Token header
export function portalClient(token?: string) {
  return createClient<Database>(SUPABASE_URL, SUPABASE_PUBLISHABLE_KEY, {
    auth: { persistSession: false, autoRefreshToken: false },
    global: {
      headers: token ? { "X-Client-Token": token } : {},
    },
  });
}
```

**Flow:**
1. Client lands on `https://my.rolliworks.com/client/:token`
2. `portal-client.ts` extracts the token from URL path: `/client/(.*)`
3. Creates a Supabase client with `X-Client-Token: {token}` header
4. Supabase RLS policies use `current_client_token()` to filter rows
5. Client can only see their own conversations, messages, and photos

### Portal pages/components

| Page/Component | Path | Purpose |
|---|---|---|
| **ClientLanding** | `pages/portal/ClientLanding.tsx` | Client home page — list of active requests by service type |
| **ClientConversation** | `pages/portal/ClientConversation.tsx` | Single conversation with message threading |
| **InspectionViewPage** | `pages/portal/InspectionViewPage.tsx` | Visual inspection results (presumably from RS) |
| **InvalidLink** | `pages/portal/InvalidLink.tsx` | Token expired/invalid |
| **PortalLayout** | `pages/portal/PortalLayout.tsx` | Layout wrapper (headers, typography) |
| **PortalHeader** | `components/PortalHeader.tsx` | Portal nav component |

### Client portal features

- **View active requests** — List of services with states: "Awaiting our reply" / "Awaiting your reply" / "In progress" / "On hold"
- **View conversation** — See all messages, photos, and status
- **Upload photos** — Clients can upload photos via the portal
- **Reply by email** — Client can reply directly to inbound emails
- **View inspection results** — Inspection results sent to client

### Welcome email flow

On new conversation creation (via `portal-intake-router`):
1. `portal-send-welcome` edge function is called (async)
2. If `email_welcome_enabled = true` and `rc_email_mode = 'live'`:
   - Loads email template (`welcome_new_client` or `welcome_returning_client`)
   - Renders with variables: `firstName`, `serviceType`, `portalUrl`, `conversationUrl`
   - Sends via Resend API
3. If in `test` mode or feature disabled → log and skip (but still mark "sent")

---

## 4. Inbound email reply system

### Inbound email flow (Postmark → RC)

1. **Email arrives at `noreply@reply.rolliworks.com`** via Postmark inbound webhook
2. **RC's `inbound-email-handler` edge function** receives the Postmark payload
3. **Token extraction** — Extracts the 16-char hex token from the email recipient address: `noreply+{token}@reply.rolliworks.com`
4. **HMAC matching** — For each of the 2000 most recent conversations, computes HMAC-SHA256(secret, conversationId) and compares with the token
5. **If matched** — Inserts the message into `messages` table, marks conversation `awaiting_team`, resets team read state
6. **If not matched** — Logs to `unmatched_inbound` table, returns `matched: false`
7. **Attachments** — Image attachments are stored in `conversation-photos` bucket; other attachments silently skipped

### HMAC reply token strategy

```
reply_token = HMAC-SHA256(
  key = Deno.env.RC_INBOUND_HMAC_SECRET,
  data = conversation.id
).slice(0, 16)
```

**Key properties:**
- **Secret is environment-configured** in RC, NOT in the database
- **Reply token = 16 hex chars** (half of a SHA-256 hash)
- **Token derivation is per-conversation** — each conversation gets a unique HMAC based on its ID
- **Token is stored on the `clients` table** `reply_token` column (unique constraint)
- **Token is used in email To field** as `noreply+<token>@reply.rolliworks.com`
- **The HMAC secret is NEVER transmitted** — only the 16-char public hash is used in email addresses
- **Verification is O(2000)** — checks all 2000 most recent conversations (not ideal, but workable)

**HMAC verification flow:**
```
1. Email arrives with recipient: noreply+abc123def4567890@reply.rolliworks.com
2. Extract token: "abc123def4567890"
3. Query 2000 recent non-archived conversations
4. For each conversation: compute HMAC(secret, conversationId).slice(0,16)
5. If matches → reply routed to that conversation
6. If no match → log to unmatched_inbound
```

**⚠️ Security consideration for rebuild:** The HMAC secret (`RC_INBOUND_HMAC_SECRET`) should be rotated if it was ever exposed anywhere. The O(2000) verification is inefficient if conversations grow beyond 2000.

### Email signature stripping

The inbound handler strips common email signatures before storing:
- Delimiter patterns: `--`, `---`, `_____`
- Device signatures: `Sent from my iPhone/Android`
- Closing signatures: `Thanks,`, `Best regards,`, etc. (only if followed by ≤5 short lines)

This prevents email footers from being inserted into the conversation.

### Signed-in-reply email handling

RC receives an email from the client. The email's reply token (embedded in the To: field) identifies the specific conversation. The system then strips email signatures and stores the reply.

---

## 5. The duplicate client problem

### Root cause

**The `clients` table has a UNIQUE constraint on `email`** (migration `20260423074336`):
```sql
CREATE TABLE public.clients (
  ...
  email text UNIQUE NOT NULL,
  ...
);
```

**But there is NO constraint on `email + phone`.** This means two different people with the same email address cannot have separate client records. More importantly, the duplicate problem likely arises from:

**Scenario 1: Same email, different names**
- A customer might have the same email but submit two different requests
- The portal-intake-router code shows: "Find or create client" logic — if email exists, it **reuses** the existing client record (not a duplicate)
- **So the duplicate is NOT from the portal** — it's from somewhere else (likely RS importing customers)

**Scenario 2: Name/email variations in RS import**
- RS customers might have name/email variations that don't match exactly
- E.g., "John Smith" vs "john smith" vs "J. Smith"
- The `portal-intake-router` does `lowercase` for comparison: `String(body.email ?? "").trim().toLowerCase()`
- But `email` is stored as-is in the table (case-sensitive) — the UNIQUE constraint is **case-sensitive** on PostgreSQL
- This means "john@example.com" and "John@example.com" would be **two different rows** despite being the same person!

**Scenario 3: No merge mechanism**
- As Mike flagged: "RC creates duplicate clients and there's no merge/split"
- The `clients` table has no `merged_into_id` or similar column
- There is **no UI** for merging duplicates (as confirmed by Mike's complaint)

**Scenario 4: Multiple RC projects**
- If RC's database is being imported from multiple sources (RS, other systems), different source systems might use different email formats
- E.g., RS stores "John Smith <john@example.com>" while another system stores just "john@example.com"
- On import, case differences create duplicates

### Where duplicates are created

1. **Portal intake** (`portal-intake-router`): If email is case-different, two rows created (no dedup check on case)
2. **Inbound email handler**: If a client with a different case sends from a new email address, creates a new row
3. **RS → RC import**: No automated import mechanism currently exists (per RS dossier: "Future RolliClient portal" is Phase 2/3)

### Current prevention mechanism

- **PostgreSQL UNIQUE constraint on `email`** prevents exact duplicates (same case)
- **`portal-intake-router`** checks `eq("email", email)` before creating → prevents duplicate portal submissions with same email
- **`uniq` index on `email`** in PostgreSQL is case-sensitive by default

### Duplicate client mitigation

**Short term:** Add case-insensitive email dedup:
```sql
-- Instead of UNIQUE(email), use a case-insensitive approach
ALTER TABLE public.clients ADD CONSTRAINT idx_clients_email_ci UNIQUE (lower(email));
-- Then lowercase all existing emails
UPDATE public.clients SET email = lower(email);
```

**Long term:** Add a merge/split mechanism:
- `merged_into_id uuid REFERENCES clients(id)` column
- UI for identifying and merging duplicate clients
- Cascade all conversations and messages to the merged client

---

## 6. Portal magic-link flow

### RC → RS magic link (client portal link)

The `portal-intake-router` creates a client token (64-char random base64url) and returns it with the conversation. The client portal URL is built as:
- **Base URL:** `https://my.rolliworks.com`
- **Client URL:** `{BASE_URL}/client/{token}`
- **Conversation URL:** `{BASE_URL}/client/{token}/c/{conversation_id}`

### Token generation

```typescript
// 64-char random token (48 bytes, base64url)
function generateToken(): string {
  const bytes = new Uint8Array(48);
  crypto.getRandomValues(bytes);
  const b64 = btoa(String.fromCharCode(...bytes))
    .replace(/\+/g, "-").replace(/\//g, "_").replace(/=+$/, "");
  return b64.slice(0, 64);
}

// 16-char hmac reply token (8 bytes, hex)
function generateReplyToken(): string {
  const bytes = new Uint8Array(8);
  crypto.getRandomValues(bytes);
  return Array.from(bytes).map(b => b.toString(16).padStart(2, '0')).join('');
}
```

### `portal-intake-router` edge function

**Endpoint:** `https://rc-project.supabase.co/functions/v1/portal-intake-router`

**Auth:** No auth required (public intake form)

**Input:**
```json
{
  "email": "client@example.com",
  "name": "John Smith",
  "phone": "555-1234",
  "service_type": "jewelry repair",
  "initial_description": "...",
  "intake_lead_id": null,
  "rs_customer_id": null
}
```

**Logic:**
1. **Find or create client** — queries `clients.by(email)`. If found, updates `last_activity_at`. If not, creates new client with `status = 'prospect'`.
2. **Create conversation** — INSERTs into `conversations` with `state = 'awaiting_team'`.
3. **Initial system message** — INSERTs system message with service type and description.
4. **Unread state** — Upserts `conversation_read_states` for the whole team.
5. **Welcome email** — Calls `portal-send-welcome` edge function (async, non-blocking).
6. **Returns** `{ client_token, conversation_id, is_new_client }`.

---

## 7. Cross-app integration (RC ↔ RS)

### RC → RS communication

| Direction | Edge function | Purpose |
|---|---|---|
| **RC → RC** | `rc-timeline-status` | Fetch live timeline status from RS |
| **RC → RS** | (none) | RC has no edge functions targeting RS directly |

**RC's `rc-timeline-status` edge function:**
- RC edge function fetches the conversation's client's `rs_customer_id`
- POSTs the `rs_customer_id` to RS's `rc-timeline-status` endpoint
- Uses `RS_TIMELINE_STATUS_KEY` for authorization
- Returns the timeline status for the client to view in the RC portal

### RS → RC communication

| Direction | Edge function | Purpose |
|---|---|---|
| **RS → RC** | `rc-portal-invite` | Send invite to RC |

The RS side has `rc-portal-invite` which sends invites to RC clients. This is how RS creates RC portal accounts for clients.

### Client matching logic (RS → RC)

When an RS customer gets an `rs_customer_id`, it links the RC `clients` record to the RS customer. The key field is `clients.rs_customer_id` — a FK to RS customers.

**Matching rule:** `client.rs_customer_id == RS customer.id`

**Gap:** `rs_customer_id` is nullable. When null, there's no link to RS. The RC timeline status function (`rc-timeline-status`) returns `{ ok: true, status: null, reason: "no_rs_customer" }` when the field is null — which is the expected behavior.

---

## 8. Nudge system

### Components

| Edge function | Purpose |
|---|---|
| `portal-send-nudge` | Send a follow-up nudge to a client |
| `portal-nudge-runner` | Scheduled cron to run nudges |
| `list-customer-approvals` | List pending approvals |

### Nudge flow

1. **`portal-nudge-runner`** runs on a schedule
2. **Finds stale conversations** (`state = 'awaiting_client'` and past a threshold)
3. **Sends `portal-send-nudge`** for each stale conversation
4. **Email template** (from `email_templates` table):
   ```
   Subject: "Just checking in — your Rolliworks request"
   Body: "Hi {firstName},
          Just checking in to see if you still need the info on your {serviceType} request.
          Whenever you have a moment, reply here:

          {conversationUrl}
          No rush — we're here whenever you're ready.
          — Rolliworks"
   ```

### Unread detection (get_inbox_unread_counts RPC)

The RPC combines two unread sources:
1. **Message-based** — Messages from clients received since the last `last_read_at` per team_user_id
2. **Manual-based** — Manually marked unread (when `marked_unread_at > last_read_at`)

Uses `GREATEST(msg_unread, marked_unread)` to avoid double-counting.

---

## 9. RLS policy summary (RC)

| Table | Role | Permissions |
|---|---|---|
| **clients** | `anon` (token) | Select own by token, update own |
| **clients** | `authenticated` | All CRUD (RLS = true) |
| **conversations** | `anon` (token) | Select/update own |
| **conversations** | `authenticated` | All CRUD (RLS = true) |
| **messages** | `anon` (token) | Select own messages, insert own (as client) |
| **messages** | `authenticated` | All CRUD (RLS = true) |
| **conversation_read_states** | `authenticated` | All CRUD (RLS = true) |
| **conversation_views** | `authenticated` | Insert own views, All CRUD (RLS = true) |
| **conversation_jobs** | `authenticated` | All CRUD (RLS = true) |
| **team_users** | `authenticated` | Read all; admins insert/update |
| **conversation_photos** | `anon` (token) | Select/insert own photos |
| **conversation_photos** | `authenticated` | All CRUD (RLS = true) |
| **storage.conversation-photos** | `anon` / `authenticated` | Path-isolated per conversation_id |
| **app_settings** | `authenticated` | Read all; admins update |
| **user_roles** | `authenticated` | Own roles; admins manage |
| **unmatched_inbound** | `authenticated` | Select all; update own |
| **outbound_attachment_hashes** | `authenticated` | All CRUD (RLS = true) |

---

## 10. RC's position in the Rolliworks ecosystem

### What RC owns

1. **Customer conversation history** — all team-client communications
2. **Customer portal** — client-side view of requests/status
3. **Client identity** — the `clients` table is RC's master customer record
4. **Email reply system** — all inbound email handling
5. **Saved replies** — team responder templates
6. **Nudge/follow-up system** — automated staleness detection

### What RC depends on

1. **RS** — Customer IDs (`rs_customer_id`), job status timeline (`rc-timeline-status`)
2. **RW** — Job inspection/test results (via RS → RC connection)
3. **Postmark** — Email delivery (via `reply.rolliworks.com` domain)
4. **Resend** — Email sending (via `portal-send-welcome` and `portal-send-nudge`)

### What RC feeds to others

1. **RS** — RC creates RC client accounts (via `rc-portal-invite`)
2. **Clients** — Welcome emails, nudge emails via Resend

### What's NOT in RC (gaps)

1. **No watch records** — Placeholder only ("Watches will appear here once RS integration is added")
2. **No estimates** — Placeholder only ("Estimates and jobs will appear here once RS integration is added")
3. **No jobs** — Placeholder only
4. **No photos** — Placeholder only ("Photos will appear here once RS/RW integration is added")
5. **No lifetime summary** — Placeholder only ("Lifetime stats will appear here once RS/QBO integration is added")
6. **No real-time RS status** — Only fetched on demand via `rc-timeline-status`
7. **No RS → RW inspection photo push** — Mike flagged this: "inspection photos can't be sent from RS to RC. Need to know what exists today before designing what's needed."

---

## 11. Rebuild recommendations

### Client matching / duplicate prevention

1. **Case-insensitive email** — Add `lower(email)` index on `clients`
2. **Phone normalization** — Add phone as secondary key
3. **Merge/split** — Implement in the CRM UI with a `merged_into_id` column
4. **RS → RC import** — When RC gets a customer from RS, check for existing clients first

### Portal security

5. **Token rotation** — Currently tokens are static. Consider allowing clients to rotate their tokens
6. **Session expiry** — Portal tokens never expire (perpetual access). Consider adding expiry or requiring re-auth

### Cross-app integration

7. **RS → RC photo push** — Implement a push mechanism so RS-generated inspection photos reach RC's portal
8. **Lazy vs eager RS customer loading** — `rs_customer_id` is optional. When loading, consider lazy-loading the RS customer data

### Inbound email

9. **Conversation lookup optimization** — Current O(2000) HMAC verification is inefficient. Use an index or hash table for O(1) lookup
10. **Token expiration** — Currently the HMAC is valid forever. Consider expiring old HMACs

### Non-functional

11. **RC has no edge functions for outbound** — All outbound communication uses Resend + Postmark. This is appropriate but means RC's edge function surface is minimal
12. **RC's storage bucket** — `conversation-photos` allows images up to 10MB. Consider limiting photo size and count per conversation
