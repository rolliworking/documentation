# RolliConnect (RC) — Context Dossier

> **Purpose:** Orientation context for downstream model work (Qwen batch jobs) on RolliConnect.
> Read this FIRST before analyzing RC code. It carries the verified schema, the live
> behavioral findings (from direct DB queries), and the rebuild priorities — so analysis
> starts grounded in what is actually happening, not just what the code can do.
>
> **Status:** Living document, partial. Schema + behavioral findings are verified from live
> queries (2026-06-07). Code-level detail is not yet filled — items marked
> `[QWEN: INVESTIGATE]` are what a code-reading pass should complete. Items marked
> `[FILL IN]` need Michael's product knowledge.
>
> **Companion docs:** `NORTH-STAR.md` (suite vision — read first), `RS_CONTEXT_DOSSIER.md`
> (RolliSuite, the source-of-truth ERP that RC relates to).

---

## 0. Verified Findings (human + live-query confirmed — read first)

> These override any conflicting inference from code-reading. Established 2026-06-07 via direct
> read-only SELECT queries against RC's production Supabase DB.

### 0.1 What RC is
RolliConnect is the **customer-facing communication system** for the watch service business —
client conversations, estimate replies, the client portal, and message threading. It is a
focused app: **15 tables total**, built around messaging (contrast RS's 118 tables).

### 0.2 Architecture: RC has its own database
RC runs on a **separate Supabase project** from RS. It does NOT share RS's database. RC relates
to RS/RW through application-level links (e.g. `intake_lead_id`, `conversation_jobs`), not a
shared DB. `[QWEN: INVESTIGATE — exactly how RC reads/writes RS data; is there an API contract,
webhook, or shared identifier?]`

### 0.3 Thread-fracturing problem — QUANTIFIED (corrects prior assumptions)
The "one client, multiple fragmented threads" problem is **real but bounded** — smaller than it
felt. Live distribution of active (non-archived) conversations per client:

| Threads per client | Number of clients |
|---|---|
| 1 | 408 |
| 2 | 49 |
| 3 | 6 |
| 4 | 2 |

- **465 distinct clients, 532 active threads → ratio 1.14 threads/client.**
- 88% of clients (408/465) have exactly one clean thread.
- Only 57 clients have ANY fragmentation; worst case is 4 threads. No client has a runaway
  count (no 10–20+ thread cases).
- `unmatched_inbound` table holds only **6 rows** — inbound message matching is working well,
  NOT leaking messages into new threads.

**Implication:** thread fracturing may be a **targeted fix** (merge ~57 clients' duplicate
threads + tighten new-conversation creation logic), not a ground-up rebuild. Conversations link
to clients by stable `client_id` (uuid), so fracturing is NOT caused by email/phone mismatch —
it comes from the app creating a *new* conversation for an existing client instead of appending.
`[QWEN: INVESTIGATE — where in the code does a new conversation get created vs. an existing one
reused? What conditions spawn a new conversation for an existing client_id?]`

- `[FILL IN — Michael: are the 57 fractured clients your frequent/high-value ones, or mostly
  dormant? Felt pain may exceed the 12% statistic if Vianna hits them often.]`
- `[FILL IN — Michael: does Vianna manually archive duplicate threads as a workaround? If so,
  the live count understates how often fracturing actually occurs.]`

### 0.4 Portal process-flow problem — NOT YET QUANTIFIED
The client portal shows a single linear "watch work" process flow. Watches in states that don't
fit it (e.g. **in-testing**, **uncased**) display confusingly. This likely affects more clients
than the threading issue (every client sees the portal), but the scale is **not yet measured**.
`[QWEN: INVESTIGATE — how does the portal decide which flow-step to show? Is the flow hardcoded
linear or data-driven? Where are watch states defined?]`
`[FILL IN — Michael: priority call — measure how many watches are currently in non-standard
states before deciding whether portal-flow or threading is the first rebuild target.]`

---

## 1. Schema — Tables (verified live)

RC has **15 public tables**:

| Table | Likely role |
|---|---|
| `conversations` | The threads themselves (schema detailed in §2) |
| `messages` | Individual messages within conversations `[QWEN: confirm FK to conversations]` |
| `clients` | Customers (RC's own client records) |
| `conversation_jobs` | Links conversations to jobs — the RC↔work link `[QWEN: INVESTIGATE]` |
| `conversation_labels` | Tagging/labeling of threads |
| `conversation_read_states` | Per-user read/unread tracking |
| `conversation_views` | View tracking `[QWEN: confirm purpose]` |
| `conversation_photos` | Photos attached to conversations |
| `messages` | Message bodies |
| `email_templates` | Outbound email templates |
| `saved_replies` | Canned responses |
| `nudge_queue` | Follow-up/reminder queue (`last_nudged_at` on conversations) |
| `unmatched_inbound` | Inbound messages that couldn't match a thread (only 6 rows — healthy) |
| `outbound_attachment_hashes` | Dedup tracking for outbound attachments `[QWEN: confirm]` |
| `team_users` | Staff users (Vianna et al.) |
| `user_roles` | Role assignments |
| `app_settings` | App config |

`[QWEN: INVESTIGATE — produce the full column schema for each of the 14 tables not yet detailed
below, especially `messages`, `clients`, `conversation_jobs`, and `unmatched_inbound`.]`

---

## 2. `conversations` table — full schema (verified live)

| Column | Type |
|---|---|
| `id` | uuid |
| `client_id` | uuid — **links to client; the fracture key** |
| `service_type` | text |
| `initial_description` | text |
| `intake_lead_id` | uuid — link to an intake lead (RS-side origin?) |
| `source_entity` | text — where the conversation originated `[QWEN: what values?]` |
| `state` | text — `[QWEN: enumerate distinct values]` |
| `completed_at` | timestamptz |
| `archived_at` | timestamptz — null = active |
| `last_activity_at` | timestamptz |
| `last_nudged_at` | timestamptz |
| `created_at` | timestamptz |
| `updated_at` | timestamptz |
| `last_message_at` | timestamptz |
| `current_status` | text — `[QWEN: enumerate distinct values; relates to portal flow?]` |
| `status_updated_at` | timestamptz |
| `status_updated_by` | uuid |
| `category` | text — `[QWEN: enumerate; does category drive separate threads?]` |

**Note for the threading investigation:** a conversation carries both `state` AND
`current_status` AND `category` AND `service_type`. A leading hypothesis for fracturing: a new
conversation is created per *job* or per *category/service_type* for the same client, rather
than threading all of a client's communication together. `[QWEN: test this hypothesis against
the code and against `conversation_jobs`.]`

---

## 3. Rebuild Priorities (from NORTH-STAR.md §5)

Two bounded RC bottlenecks, both byproduct sources. **Priority order is now under review given
§0.3 findings.**

1. **Client portal process flow** — handle non-linear states (in-testing, uncased) and
   multiple/returning-client flows. *Byproduct:* accurate client-facing status = trust asset.
   Scale not yet measured (§0.4). Wishlist refs: W14, W17.
2. **Message thread fracturing** — now quantified as bounded (§0.3). May be a targeted merge +
   logic fix rather than a rebuild. Wishlist refs: W10, W16, W17.

`[FILL IN — Michael: after measuring portal-flow scale, confirm which is the first rebuild
target. Current evidence leans toward portal-flow being broader-impact.]`

---

## 4. What Qwen should produce next (suggested batch tasks)

> Bounded, read-only analysis tasks. One module/area at a time.

1. **Full schema dump** — column detail for all 15 tables (complete §1/§2).
2. **Conversation-creation logic** — find every code path that creates a `conversations` row;
   document the conditions. This directly answers the fracture root cause (§0.3).
3. **Portal flow logic** — find how the client portal renders the process flow; determine if
   it's hardcoded-linear or data-driven; locate where watch/conversation states map to display
   steps (§0.4).
4. **Workflow + gap doc** for each of the two priority areas, using the proven template
   (Purpose / Key files / Current behavior / Inputs-outputs / Dependencies / What's mock vs
   real / Open questions) — same template validated on RolliTime.

---

## 5. Open questions index

- §0.2 — RC↔RS data link mechanism `[QWEN: INVESTIGATE]`
- §0.3 — Are fractured clients frequent/high-value? `[FILL IN — Michael]`
- §0.3 — Does Vianna archive duplicates as workaround? `[FILL IN — Michael]`
- §0.3 — Conversation-creation conditions in code `[QWEN: INVESTIGATE]`
- §0.4 — Portal flow: hardcoded vs data-driven `[QWEN: INVESTIGATE]`
- §0.4 — Count of watches in non-standard states `[FILL IN — Michael: run query]`
- §1 — Full schema for 14 remaining tables `[QWEN: INVESTIGATE]`
- §2 — Distinct values of `state`, `current_status`, `category` `[QWEN: INVESTIGATE]`
- §3 — First rebuild target confirmation `[FILL IN — Michael]`
