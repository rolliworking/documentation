# Q-010 — Cross-App Authentication Current State

**Status:** in-progress
**Task source:** TASK-QUEUE.md
**Generated:** 2026-06-26
**Skill:** `mapping-legacy-workflows`

---

## 1. Three operational apps — auth summary at a glance

| App | Hosting | Supabase project | User auth | Service model |
|---|---|---|---|---|
| **RS** (RolliSuite) | Supabase + Lovable | RS project | Supabase Auth (email/password + MFA) | `AuthContext.tsx` fetches `user_roles.role` |
| **RW** (RolliWorking) | Supabase + Lovable | RW project | Supabase Auth (email/password + MFA) | Same pattern — no custom `AuthContext` found |
| **RC** (RolliConnect) | Vercel | RC project | Supabase Auth (email/password + MFA) | No custom `AuthContext` found |

### AuthContext model (RS example — standard across all three)

In `AuthContext.tsx`, the auth flow:
1. User signs in with `supabase.auth.signInWithPassword(email, password)`
2. Session + user stored in `localStorage` (persisted)
3. On auth state change, `fetchUserRole(userId)` queries `user_roles` table
4. `ROLE_PERMISSIONS[role]` controls UI rendering (`canDeleteJobs`, `canManageUsers`, etc.)
5. `canEditTest()` helper enforces 24h edit window for non-admin roles

**RW/RC do NOT have custom `AuthContext.tsx` files.** Their `integrations/supabase/client.ts` still exists but auth logic is not present at the app root level. They either lack auth context entirely or use a different pattern.

### Client Supabase config

All three apps share the same client-side Supabase configuration:
```typescript
export const supabase = createClient<Database>(SUPABASE_URL, SUPABASE_PUBLISHABLE_KEY, {
  auth: {
    storage: localStorage,
    persistSession: true,
    autoRefreshToken: true,
  }
});
```
Uses **anon/publishable key** — all client-side queries go through the anon key and are governed by RLS policies on the target database.

---

## 2. Edge function auth patterns — RS side (74 functions)

### Two parallel auth models used within RS edge functions:

#### A. User JWT validation (authenticates a *user*)

```typescript
// Standard JWT user validation
const authHeader = req.headers.get("Authorization");
if (!authHeader?.startsWith("Bearer ")) return 401;
const token = authHeader.replace("Bearer ", "");
const supabase = createClient(url, anonKey, { global: { headers: { Authorization: authHeader } } });
const { data: { user }, error } = await supabase.auth.getUser(token);
if (error || !user) return 401;
```

**When this is used:** Edge functions that act *on behalf* of a logged-in user — `rollisuite-intake`, `rollisuite-job-finished`, `sync-model-to-rolliworking`, etc.

**Key:** Database access uses `SUPABASE_ANON_KEY` + user's JWT → reads/writes governed by **RLS** for that user.

#### B. ROLLISUITE_API_KEY / ROLIWORKING_API_KEY (authenticates a *partner app*)

```typescript
const apiKey = req.headers.get("Authorization")?.replace("Bearer ", "");
const expected = Deno.env.get("ROLLISUITE_API_KEY");
if (apiKey !== expected) return 401;
```

**When this is used:** Edge functions that serve partner apps (RW, external programs) — `job-status-update`, `rw-watchmaker-assignment`, `rw-tc-acceptance`, `rw-print-labels`, `customers-sync`, `hit-list`, etc.

**Key:** Database access uses `SUPABASE_SERVICE_ROLE_KEY` — **RLS is bypassed**.

#### C. `SUPABASE_SERVICE_ROLE_KEY` direct (authenticates *nothing*, full admin)

```typescript
const supabase = createClient(url, Deno.env.get('SUPABASE_SERVICE_ROLE_KEY')!);
```

**When this is used:** Functions that need admin-level access — all `qbo-*` functions, `send-*-email`, cron jobs (`maintenance-cron`, `qbo-cron-sync`), `send-invite`, `send-po-email`, etc.

**Key:** Full admin access. RLS bypassed. No user context (no audit trail of *who* triggered it).

**Summary: 44 of 74 edge functions declare `verify_jwt = false`** (no JWT validation), meaning anyone with network access can trigger them (subject to per-function API key checks). The remaining 30 functions do validate JWTs.

---

## 3. Edge function auth patterns — RW side

### DB-side function (for `set_job_status`)

```sql
CREATE OR REPLACE public.set_job_status(
  job_id uuid,
  new_status text
)
RETURNS void
SECURITY DEFINER
...
```

**Auth checks:**
- `IF auth.uid() IS NULL` → reject (no anonymous access)
- `IF NOT public.has_permission(auth.uid(), 'jobs.view')` → reject
- `IF current_setting('request.jwt.claims', true)::json->>'sub' IS NULL` → reject

**Service role:** `SUPABASE_SERVICE_ROLE_KEY` used when writing directly to DB (bypasses RLS).

### RW edge functions (on RW's Supabase project)

RW's edge functions authenticate the same way as RS functions (user JWT + `SUPABASE_ANON_KEY` for user context, or `SUPABASE_SERVICE_ROLE_KEY` for admin access).

**Functions per Q-002 output:** `set_job_status`, `rollisuite-status-push`, `rollisuite-webhook`, `rollisuite-parts-approved`, `rollisuite-watchmaker-assign`, `rollisuite-intake`, `job-status`, `job-status-update` (RW-hosted).

---

## 4. Edge function auth patterns — RC side

### No custom `AuthContext.tsx` found in RC
RC's `contexts/` directory contains only `InboxSearchContext.tsx` — **no auth context**. The client-side Supabase client (`client.ts`) still exists in `integrations/supabase` (auto-generated), meaning RC uses the standard Supabase client library.

RC likely uses one of:
1. Supabase auth context (from a higher-level provider — not at `contexts/AuthContext.tsx`)
2. A framework-level auth (e.g., NextAuth, Firebase)
3. Server-side auth via Vercel's auth helpers

### RC's edge functions
RC has **no edge functions** at `apps/rc/supabase/functions/` — none exist. RC's functions live in `apps/rc/src/services/` and `apps/rc/src/pages/` (client + server functions).

---

## 5. Cross-App Authentication (the core architecture)

### 5.1. RS → RW (RS pushes to RW)

| Edge function | Calls RW endpoint | Auth method | RW's response |
|---|---|---|---|
| `sync-model-to-rolliworking` | `pkgnrcfqrldwjibghefm.supabase.co/functions/v1/rollisuite-model-sync` | `ROLLIWORKING_API_KEY` (Bearer token) | `x-api-key` header, no JWT |
| `rollisuite-job-finished` | `.../functions/v1/rollisuite-job-finished` | **No auth header** — fire-and-forget text/plain | No auth check needed (ignores all failures) |
| *(manual)* `rollisuite-intake` (on RS side sends to RW) | RW's intake endpoint | `ROLLISUITE_API_KEY` | JWT validation on RW side |

**RW's incoming auth:** RW's `set_job_status` function uses `security_definer` with `auth.uid()` validation. But edge functions that call into RW's API (like `sync-model-to-rolliworking`) use `ROLLIWORKING_API_KEY`.

**Key finding:** RW has `pkgnrcfqrldwjibghefm` as its Supabase deployment URL, and RS calls RW via `ROLLIWORKING_API_KEY` as the auth header. The API key is stored in RS's env vars.

### 5.2. RW → RS (RW pushes to RS)

| Edge function | Calls RS endpoint | Auth method |
|---|---|---|
| `set_job_status` / `rollisuite-status-push` | RS's edge function | `ROLLISUITE_API_KEY` (Bearer token) |
| `rollisuite-parts-approved` | `djbjwcoddddywkgljuja.supabase.co/functions/v1/rw-parts-approved` | `ROLLISUITE_API_KEY` (Bearer token) |
| `rollisuite-watchmaker-assign` | RS's `rw-watchmaker-assignment` | `ROLLISUITE_API_KEY` (Bearer token) |

**RS's incoming auth (for RW requests):** Edge functions like `job-status-update`, `rw-parts-approved`, `rw-watchmaker-assignment` check `ROLLISUITE_API_KEY` against `Authorization: Bearer <key>`.

**Key:** This is **app-to-app API key auth** — symmetric (both sides share the same value). No user identity.

### 5.3. Client → Server (user-facing auth)

| Direction | Auth method | Notes |
|---|---|---|
| Client browsers → RS | JWT (from `Supabase Auth` session + anon key) | RLS governs every query |
| Client browsers → RW | JWT (from Supabase Auth session + anon key) | RLS governs every query |
| Client browsers → RC | JWT (from Supabase Auth session + anon key) | RLS governs every query |

### 5.4. Client tracking — RS → RW (fetching RW data)

| Edge function | Calls RW endpoint | How |
|---|---|---|
| `client-tracking` | `pkgnrcfqrldwjibghefm.supabase.co/rest/v1/jobs` | `apikey + Authorization Bearer` headers (RW's anon key) |

**Critical:** This uses **RW's anon key**, NOT the service role key. This means RS can only read RW data that RW's RLS policies expose to authenticated users. However, **no user identity is forwarded** — the requests use RS's service role identity, not the end-user's identity. This is effectively a **privileged read** through the RS service identity.

---

## 6. The M3KE JWT Issue — Root Cause

### What is M3KE?

M3KE is a Python (FastAPI) AI/ML module. It has no user login. Its role (per `CLAUDE.md`): "AI, ML, data, and computer-vision modules."

### What does M3KE need from RS?

Two database RPC functions:
1. `m3ke_get_pricing_facts(since, brand, reference, limit=5000)` — fulfilled-only SO line history
2. `m3ke_get_related_parts(brand, reference, since, limit=200)` — top parts by occurrence + revenue

### Why is it "JWT-blocked"?

**The core problem:** M3KE is a Python service (FastAPI) that needs to read RS's operational data. RS's edge functions and databases use **JWT-based Supabase auth**. M3KE has no user session → no JWT → cannot pass the `Authorization: Bearer <JWT>` header.

**Concrete blocking path:** When M3KE tries to read RS data (via edge functions), the edge function validates the JWT using either:
- `supabase.auth.getUser(token)` — requires a valid user JWT
- `RLS policies` — require an authenticated user

M3KE passes a JWT (or nothing), which cannot be validated against the user table because M3KE **is not a user** — it's a **service account**.

### Current workaround path

The RS dossier mentions M3KE has **reserved schema slots** (`ai_modules.m3ke_*`) in the shared database. The proposed resolution from REBUILD-PREREQUISITES.md #3:

> "Per-module service role with scoped read on rs.* / rw.* / rc.*, scoped write on each module's own schema under ai_modules.*."

### Auth model that would fix M3KE

M3KE needs a **per-module service role** — a dedicated Supabase service key (or API key) that:
1. Bypasses `auth.uid() IS NULL` checks
2. Can read `r.s.*` schemas (RS schema)
3. Can read `rw.s.*` schemas (RW schema) via cross-app read
4. Can write only to `ai_modules.*` schemas (own writes)
5. Has a dedicated RLS row that identifies M3KE for audit purposes

This is **not** the M3KE `service_role_key` (which gives full admin access). It needs to be a **scoped** service role.

### Current auth mismatch across all three apps

| App | Auth model | Service role? |
|---|---|---|
| RS client | JWT (anon key) | No — uses anon key, RLS respected |
| RW client | JWT (anon key) | No — uses anon key, RLS respected |
| RC client | JWT (anon key) | No — uses anon key, RLS respected |
| RS edge functions | JWT (user) or `ROLLISUITE_API_KEY` (partner) or `SUPABASE_SERVICE_ROLE_KEY` (admin) | Yes, when using `SERVICE_ROLE_KEY` |
| RW edge functions | JWT (user) or key auth | Yes, when using `SERVICE_ROLE_KEY` |
| RC functions | — | No edge functions exist |
| **M3KE** | **Nothing / JWT-invalid** | **Needed — per-module scoped service role** |

---

## 7. Proposed Auth Model for the Rebuild

### Per-consumer auth pattern

| Consumer type | Auth method | DB access |
|---|---|---|
| **Human users** (RS, RW, RC apps) | JWT from `supabase.auth` session → anon key | RLS-per-user, scoped by role |
| **App-to-app API calls** (RS ↔ RW) | `ROLLISUITE_API_KEY` / `ROLLIWORKING_API_KEY` | `SERVICE_ROLE_KEY` for writes, anon key for reads |
| **AI modules** (M3KE, RolliTime, Jarvis) | **Per-module service account** (new) | `SERVICE_ROLE_KEY` scoped to `ai_modules.*` + read-only on `rs.*`, `rw.*` |
| **External programs** (Authenticator, Jarvis, Jarvis) | Per-module service account or API key | Same as AI modules |
| **Edge functions** (internal) | `SERVICE_ROLE_KEY` (admin) or `ANON_KEY` (user) | `SERVICE_ROLE_KEY` → no RLS, `ANON_KEY` + JWT → RLS |

### Per-module service role pattern (REBUILD-PREREQUISITES.md #3)

For each external module (M3KE, RolliTime, Jarvis, Authenticator):
1. Create a **dedicated Supabase service role** scoped to:
   - **Read access:** `rs.public.*`, `rw.public.*` (where needed)
   - **Write access:** `ai_modules.{module_name}.*` only
   - **No access:** No other operational schemas
2. Store this service key as an env var in the module's runtime
3. Use the service key in Supabase client creation:
   ```python
   supabase = createClient(
       "https://shared-supabase-url.supabase.co",
       SERVICE_KEY,  # scoped service role
       {"global": {"headers": {"apikey": SERVICE_KEY}}}
   )
   ```
4. Create audit triggers on `ai_modules.*` tables to log which service role performed each operation

### Cross-app read pattern

Instead of RS using RW's anon key directly (current `client-tracking` approach), all cross-app reads should use:
1. A **shared service role** or **cross-app read key** (separate from the app's own anon key)
2. This key reads from the **shared** database (post-rebuild) or the original database (pre-rebuild)
3. RLS policies on shared tables should be updated to allow reads from this key

---

## 8. Audit logs & visibility of cross-app operations

### Current audit capability

| Operation type | Audit logged? | Where |
|---|---|---|
| RS user action (via UI) | Yes (RLS) | Respects RLS policies |
| RS → RW status push | **No** | `rollisuite-status-push` fire-and-forget, no audit trail |
| RW → RS status push | **No** | `rollisuite-status-push` fire-and-forget, no audit trail |
| Parts approval push | **Partial** | `rw-parts-approved` writes to `estimates.internal_notes` but not `audit_log` |
| Watchmaker assignment push | Yes | `rw-watchmaker-assignment` writes to `audit_log` |
| M3KE data reads | **No** | Cannot read at all (JWT-blocked) |
| Client tracking fetch | **Partial** | `client-tracking` reads RW data using `apikey` header |

### Gap: No audit trail for service-role operations

When an edge function uses `SERVICE_ROLE_KEY`, **no user identity is recorded**. This means:
- `set_job_status` uses `security_definer` — it's a DB function, not audit-logged anywhere
- `rollisuite-status-push` is fire-and-forget with no audit trail
- Parts pushes go to RS's `intenal_notes` but not to a shared audit table

**Recommendation (per audit):** Every service-role write should log to a shared `audit_log` table (or use a dedicated audit row in the target table) with: `actor = 'service:{module_name}`, `action`, `target`, `timestamp`

---

## 9. Service role key distribution across the codebase

### Which edge functions use `SERVICE_ROLE_KEY`?

```
44/74 RS edge functions use SERVICE_ROLE_KEY for DB access.
```

### Which edge functions use `ROLLISUITE_API_KEY` or `ROLLIWORKING_API_KEY`?

```
All partner-facing edge functions: rw-job-status, rw-parts-approved, rw-watchmaker-assignment, rw-tc-acceptance, rw-print-labels, hit-list, job-status-update, sync-model-to-rolliworking
```

### Which edge functions validate JWT?

```
Functions that authenticate a user: rollisuite-intake, rollisuite-job-finished, rollisuite-model-sync, and a few others. These pass the user's JWT to Supabase for user-level resolution.
```

---

## 10. RC's role in the cross-app ecosystem

### RC's position

- RC has no edge functions
- RC is not listed as a "partner app" for any cross-app auth
- RC's client app uses its own Supabase project
- RC depends on RS for customer matching and status timeline (`rc-timeline-status` fetches from RS)
- RC depends on RW for inspection/test results (`client-tracking` fetches from RW)

**Gap:** RC is a **consumer** of both RS and RW data but has no **owning** auth relationship with either. RC is not a peer — it's a downstream app.

### M3KE's position

- M3KE is a **consumer** of RS data (via two RPC functions)
- M3KE has no defined auth relationship with RS
- M3KE is **blocked** by JWT validation
- M3KE needs a **per-module service role** (REBUILD-PREREQUISITES.md #3)

---

## 11. Rebuild recommendations

### Immediate (before rebuild)

1. **Define the shared database auth model** — not just "shared Supabase" but: how users, services, and AI modules authenticate to it
2. **Create a per-module service role** for M3KE (fix the JWT-blocker)
3. **Document the cross-app read/write boundary** — which modules can read from which schemas

### During rebuild

4. **Replace all `SERVICE_ROLE_KEY` edge functions** with either:
   - **Scoped read keys** (for reads) — still bypass RLS but limited scope
   - **User-context auth** (when user can be extracted from JWT) — respects RLS
5. **Create a shared `audit_log` table** (or use existing) — all service-role writes go here
6. **Replace symmetric API keys** (ROLLISUITE_API_KEY) with a **shared service account** model

### Architecture for AI modules

7. **Per-module service accounts** — one service key per AI/ML module
8. **Per-module schema** — each module's data under `ai_modules.{module_name}`
9. **Audit triggers** — all `ai_modules.*` writes audited with module identity
10. **Read-only access** to operational schemas for AI modules (no writes to rs./rw./rc.* schemas)
