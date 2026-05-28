# Rolliworks Three-System Architecture + AI Assistant — Handoff for New Chats

**Last updated:** May 12, 2026
**Owner:** Jackson Kim, Rolliworks Inc
**Purpose:** Onboard a fresh Claude chat to the current state of the Rolliworks software ecosystem (RS, RW, RC) and the plan to introduce a local AI workstation as a job-lookup assistant. When this doc and reality disagree, this doc is the intent — reality is what's been built so far.

---

## TL;DR — Read this first

Rolliworks Inc is a high-end watch repair business: ~5-17 service requests/day, average ticket $800-1,000, $1.5M annual revenue. Single jobs can exceed $30K. We're a small high-margin business, not a scaling-to-millions SaaS. **Quality per request matters infinitely more than throughput.**

Three Lovable codebases that talk to each other:
- **RS (RolliSuite)** — internal ERP. Customer identity, leads, estimates, QBO sync.
- **RW (RolliWorking)** — workshop floor management. Jobs, components, shop floor map.
- **RC (RolliConnect)** — client portal + team-facing CRM. The communication layer.

Plus a planned **Corsair AI Workstation 300** (Ryzen AI Max+ 395, 128GB unified RAM, ~96GB usable as VRAM) running local LLMs (Gemma 4, possibly OpenAI GPT-OSS 120B) to serve as an internal AI assistant for job lookup, status queries, and structured-output tasks across the three systems.

Current state: Planning is complete for Phase 1. RC's Wix-chat-replacement sprint started May 12 (tonight). Architecture contract between the three systems is locked at v2 with three known gaps deferred to Phase 2.

---

## 1. The business

### What Rolliworks does
High-end watch repair and restoration. Clients send Rolex, Patek Philippe, vintage pieces. Service codes use W- (Watchmaker), BR- (Bracelet/Band Room), POL- (Polish), PM- (PM band).

### Volume reality
- ~5-17 service requests/day (5 was the baseline estimate, 17 was observed on May 12)
- ~$800-1,000 average ticket
- Single jobs up to $30K+
- A "crazy busy" shop would be 30/day = $10-15M revenue/year
- Postgres + Supabase + standard web app patterns scale to 1,000/day without architectural changes. **No premature optimization needed.**

### Why this matters for architecture
- Each lost client costs more than weeks of engineering work
- Personal touch is a competitive advantage
- Manual review of every request is viable and desirable
- Build for fit and finish, not for volume
- Optimize for the high-end client (Patek Philippe owner), not the median (battery change)

---

## 2. The three systems

### RS — RolliSuite (ERP, the front of the funnel)

**Owns:**
- Customer identity (`customers` table)
- Intake leads (`intake_leads`)
- Estimates, sales orders, appointments
- Shipping label requests, package arrival scans
- QBO (QuickBooks Online) sync
- The Wix intake webhook (`wix-intake-webhook`)

**Connects to:** Wix form on rolliworks.com (inbound), QBO (sync), RW (Stage 3 handoff), RC (intake handoff to lead's portal).

**Three-stage intake model:**
1. **Stage 1 — Estimate created:** Service codes only, no physical confirmation. Data lives in RS + QBO.
2. **Stage 2 — Ship intake/drop-off:** Receipt buttons set `received_bracelet`, `received_case`. Orphan conditions detected. Nothing sent to RW yet.
3. **Stage 3 — Receive watch:** Watchmaker opens entry. Service codes locked, flow type computed, orphan parking decided. Labels print. Payload posts to RW's `rollisuite-intake` endpoint. RW creates `jobs` + `job_components` rows from this single message.

**Key columns on `intake_leads`:**
- `id`, `received_at`, `email`, `full_name`, `phone`, `notes`
- `watch_reference`, `watch_serial`, `item_type`, `insured_value`
- `rc_conversation_id`, `rc_client_token` (links to RC)
- `rc_sync_failed_at`, `rc_sync_error_message` (added May 12 in atomic-swap sprint)

**Key edge functions:**
- `wix-intake-webhook` — receives Wix submissions, dedupes by `email_normalized`, creates `intake_leads`. Currently fires inline call to RC (being refactored May 12).
- `rs-intake-from-rc` — NEW (May 12). Takes `{ lead_id }`, hydrates from `intake_leads` + `customers` + `watches`, POSTs to RC's `portal-intake-router`.
- `rollisuite-intake` (note: lives on RW, called by RS) — Stage 3 handoff to RW.

**Known issues:**
- `wix-intake-webhook` inbound auth is URL obscurity only. Phase 2 hardening needed.
- `intake_leads.email_normalized` dedupe is case/whitespace sensitive. Typos create duplicates.
- `source_entity` column doesn't exist. Hardcoded "rolliworks" for now.

### RW — RolliWorking (workshop, the middle of the funnel)

**Owns:**
- Jobs (`jobs` table, UUID-keyed)
- Job components (`job_components` table)
- Two-lane shop floor workflow (head lane + band lane, converge at final assembly)
- Watchmaker assignments, parts approvals, status transitions
- Job status audit (`job_status_changes` table)
- Existing client-reply mechanisms: `ClientReplies` page, `NoReplyInbox` (for parts approvals, waivers, custom notes — these will eventually move to RC, but not in Phase 1)

**Connects to:** RS (inbound via `rollisuite-intake`), RC (planned Phase 2 via `rc-job-status`).

**Job status lifecycle (8 states):**
`waiting_approval → in_queue → in_progress → parts_approval → parts_on_order → in_testing → waiting_components → finished`

Plus component-level states: `received`, `in_progress`, `in_safe`, `waiting_components`, `reunited`, `finished`, possibly `cancelled` (TBD).

**Shop floor map (two-lane visual):**
- **Head lane:** Assign watchmaker → Uncase → 🔒 → Assign refinisher → Refinish case → 🔒 → Movement service → Parts approval → Parts on order → In testing → Recase + test → 🔒
- **Band lane:** Band tech → 🔒 → Assign refinisher → 🔒 → Refinish bracelet → 🔒 → QC inspect → 🔒
- **Converge:** Await head → Final assembly → Done

🔒 = "safe" / queue state where components physically rest between active stations.

**`rollisuite-intake` response shape (stable, confirmed May 12):**
```json
{
  "success": true,
  "customerId": "uuid",
  "watchId": "uuid",
  "jobId": "uuid",
  "services": ["..."],
  "serviceType": "...",
  "targetWeeks": 4,
  "dueDate": "YYYY-MM-DD",
  "flow": "STANDARD_FLOW",
  "message": "Intake data processed successfully"
}
```

**Critical correlation key insight:** `estimate_number` is NULLABLE TEXT and is NOT UNIQUE across multi-item estimates. Use `jobs.id` (UUID) as the canonical correlation key. RC stores `rw_job_id` for this reason.

**Two-lane state is INFERRED, not stored.** No "station" or "location" column exists. Canonical lane state = `(job_components.department, job_components.status, status_changed_at)` per component, with `jobs.flow` indicating whether the band lane exists.

**Component status enum is inconsistent between DB and code** (e.g., `reunited`, `fulfilled`, `cancelled` written by app but not in live values). When RC pulls from RW in Phase 2, RW must normalize server-side to RC's display enum.

**Known issues:**
- All RW inbound endpoints rely on URL obscurity for auth. Phase 2 hardening needed.
- `jobs.assigned_watchmaker` (text) and `job_components.assigned_watchmaker_id` (uuid) can disagree. Component-level is truth.
- `jobs.updated_at` is noisy (updates on any DB change). For "true client-visible status timestamp" use `job_status_changes` audit table instead.
- `job_status_changes` does NOT have `estimate_number`. Foreign key is `job_id` only. Currently has single-column indexes on `job_id` and `changed_at` — composite `(job_id, changed_at DESC)` index recommended for Phase 2.

### RC — RolliConnect (CRM + client portal, the communication layer)

**Owns:**
- Conversations (`conversations`)
- Conversation-to-job links (`conversation_jobs`, 1:many)
- Clients mirror (`clients` — mirrored from RS)
- Messages, photos (`conversation_photos`)
- Saved replies (per-user, RLS isolated)
- Team users, roles, read state
- Magic-link client portal auth
- Email orchestration (Resend, `quotes.rolliworks.com` outbound)

**Connects to:** RS (inbound intake, planned `rs-customer-promoted` and `rs-job-created` pushes), RW (planned Phase 2 `rc-job-status` pull).

**Conversation ↔ Jobs model:** 1:many. One conversation per Wix submission. Can link to 0-N RW jobs.
- 0 jobs: pre-estimate/lead stage
- 1 job: typical single-watch repair
- N jobs: multi-item estimate (Phase 2 UI tabs)

**Key edge functions:**
- `portal-intake-router` — receives intake from RS (was Wix→RC direct, now Wix→RS→RC as of May 12). Dual-auth window during transition (accepts `Bearer` AND `x-api-key`).
- `portal-send-welcome` — auto-reply email with magic link
- `portal-send-reply-notification` — notifies client when team replies
- `portal-send-nudge` — automated follow-ups
- `portal-nudge-runner` — scheduled cron for nudges

**Phase 1 Wix-killer sprint (in progress, May 12 onward):**
| Night | Feature |
|---|---|
| Mon (May 12) | Wix → RS → RC intake routing + auto-reply with magic link |
| Tue | Outbound email Pattern Y (full message content vs notification-only) |
| Wed | Postmark Inbound setup, DNS, HMAC reply-to tokenization (3hr night) |
| Thu | Inbound text replies + photo attachments → RC conversations |
| Fri | Embedded photo attachments in saved replies (status badge punts to week 2) |

**Sprint goal:** Retire Wix chat. RC handles new-lead intake, message threads, email round-trips. NOT unified client communication (RW's `ClientReplies` stays untouched in Phase 1).

**Already built (pre-sprint):**
- Prompt 1: Tables, routes, CRM + portal shells, seed data, `portal-intake-router`
- Prompt 2: Email sending (welcome, reply notifications, nudges)
- Saved replies: per-user table with RLS, horizontal carousel UI, modal create/edit/delete
- `conversation_photos` table + PhotoUpload/PhotoGallery/PhotoLightbox/InlinePhotos components
- `app_settings.rc_email_mode` (test/live) with `EmailModeBadge`
- Global client search (`CrmHeaderSearch` + `/crm/search`)

**Not yet built (Phase 2+):**
- Live shop floor map (placeholder currently — "Coming Soon")
- Vertical pre-shop-floor tracker (Quote request → Information Gathering → Quote sent → branching paths → Item received → Inspection)
- Multi-item folder/tab view
- Parts approval routing through portal
- Intake/inspection photos display
- Inspection notes download
- RS booking link + Request photos buttons in composer
- Global control panel (on/off toggles for portal display)
- Custom domains (`crm.rolliworks.com`, `my.rolliworks.com`)
- Mobile responsiveness pass

---

## 3. Cross-system integration contract (v2)

**Data flow:**
```
Wix form → RS wix-intake-webhook → RS rs-intake-from-rc → RC portal-intake-router
                                                        ↓
                            (At Stage 3) RS → RW rollisuite-intake
                                                        ↓
                            (Phase 1.5, not built) RS → RC rs-job-created
                                                        ↓
                            (Phase 2, not built) RC pulls from RW rc-job-status
```

**Auth model:**
- Every NEW endpoint validates `x-api-key` header
- Symmetric secret naming: each system has `<OTHER>_INBOUND_KEY` (verifies) and `<OTHER>_OUTBOUND_KEY` (sends)
- Pairing rule: `A._OUTBOUND_KEY` value === `B._INBOUND_KEY` value
- RC supports dual-auth during May 12 cutover (legacy `Bearer` + new `x-api-key`), removes Bearer after 24h confirmation

**Idempotency:**
- `rs_lead_id` is the upsert key on RC's `portal-intake-router`
- `(rs_lead_id, rw_job_id)` is the upsert key on RC's planned `rs-job-created`
- RC never writes to RS or RW. Read-only.

**Three known v2 contract gaps (deferred to Phase 2):**
1. Portal access timing — v2 says "issued at promotion." Reality is portal access at intake. The contract is wrong; code follows the correct behavior. Re-cut v3 someday.
2. `waiting_approval` enum disambiguation — both `received` and `awaiting_estimate_approval` map from `waiting_approval`. Needs a one-sentence rule.
3. Component-level data in `rc-job-status` — should return component-level statuses in addition to `overall_status` for team CRM consumption. Not yet specified.

**Known pre-existing risks (tracked, not Phase 1 scope):**
- RS and RW inbound endpoints rely on URL obscurity
- Wix duplicate-submission creates duplicate `intake_leads`
- No push events from RS/RW to RC for status changes (pull-only Phase 1)
- Split-brain client communication: RW's `ClientReplies` runs independently of RC for Phase 1

---

## 4. The AI Workstation — planned addition

### Hardware

**Corsair AI Workstation 300** with the high-end SKU:
- AMD Ryzen AI Max+ 395 (16-core, 32-thread Zen 5, 3 GHz base / 5.1 GHz boost)
- 128GB LPDDR5X-8000 unified memory
- Radeon 8060S iGPU with up to 96GB VRAM (unified memory architecture — dynamically allocated)
- XDNA 2 NPU delivering up to 50 TOPS
- 1-4TB NVMe SSD
- 4.4L compact aluminum chassis
- Built-in 350W PSU
- Wi-Fi 6E, 2.5G Ethernet
- Performance level selector (Quiet ~55W / Balanced ~85W / Max ~120W)

**Pricing reference:** Started at $1,999 (128GB + 1TB) when first released, ~$2,299 for higher-storage configs.

**What makes this hardware important for our use case:** The 96GB usable VRAM via unified memory allocation means **large local models that won't fit on a 32GB RTX 5090** run smoothly here. The Radeon 8060S can run Meta Llama 4 Scout 109B (92GB at 8-bit), or OpenAI GPT-OSS 120B (~63GB with MXFP4 quantization) without thermal throttling.

### Use cases for the three Rolliworks apps

**Primary use case: internal AI assistant for job lookup and info**

A natural-language interface the team can use to query across RS, RW, and RC without writing SQL or clicking through multiple UIs. Examples:

- "What's the status of the Submariner for Jim Smith?"
- "How many jobs are waiting on parts approval right now?"
- "Show me all jobs in the band lane that have been there more than 5 days"
- "What did Mike say about the Patek that came in last Tuesday?"
- "Which clients have requests older than 7 days with no team reply?"
- "Pull up everything we have on the Datejust we shipped Monday"

The assistant would have read-only access to all three databases via Supabase service-role keys, plus possibly read access to RC's conversation history.

**Architecture for this use case:**

```
Team member (browser or chat UI)
    ↓ natural language query
AI Workstation API endpoint
    ↓ LLM interprets query + decides what to fetch
Supabase queries to RS, RW, RC (read-only)
    ↓ structured data returned
LLM formats response in natural language
    ↓
Team member sees answer + supporting links
```

The team-facing UI could be a simple chat interface in RC's CRM (a new "Ask AI" tab), or a separate lightweight web app pointed at the workstation.

**Secondary use cases (described in earlier planning, ranked by leverage):**

1. **Quote-stripping on inbound emails (RC, Phase 1.5)** — when clients reply to RC emails, strip signatures and quoted history to extract just the new content. ~4-6 hour build to integrate. High value.

2. **Watch identification from intake photos (RW, Phase 2/3)** — multimodal LLM looks at intake photos and pre-populates brand, model, reference, condition flags. ~10-15 hour build. Very high value for client portal display ("We've received your two-tone Rolex Submariner 116610LN").

3. **Bracelet identification from poster (RW, Phase 2)** — using the bracelet identification poster as a reference, ID a client's bracelet from photos. ~6-8 hour build.

4. **Estimate prep from raw client description (RS, Phase 2)** — extract structured service-code suggestions from free-text Wix submissions. ~6-10 hour build.

5. **Inspection note structuring (RW, Phase 3)** — translate watchmaker shorthand into client-friendly portal copy. ~3-5 hour build.

6. **PII/sensitive data redaction (cross-system, Phase 3)** — scrub serial numbers and addresses before logs or external API calls. Privacy hygiene.

**Use cases NOT recommended:**

- **Replacing existing Gemini OCR** for inspection forms and handwriting. Gemini's accuracy on those tasks is ~95-98%; Gemma 4's would be ~70-85%. Gemini cost at our volume is negligible (~$3-30/month). The accuracy gap matters for production-critical OCR.
- **Anything client-facing that auto-sends without human review.** AI hallucinations + client trust = bad combination.
- **Anything in the critical path of money flow** (estimates, invoices). Human review only.

### Software stack on the workstation

Recommended:
- **OS:** Ubuntu 24.04 LTS (more reliable than Windows for 24/7 inference workloads, better Ollama support, lower overhead)
- **Inference server:** Ollama (or LM Studio for GUI experimentation)
- **Models:** Gemma 4 E4B (general purpose, multimodal), possibly OpenAI GPT-OSS 120B (heavier, for complex reasoning), Llama 4 Scout if needed for very long context
- **API layer:** FastAPI or similar Python service wrapping Ollama with auth + rate limiting
- **Reverse proxy:** Caddy or Nginx with auto-TLS for exposing the workstation to Supabase edge functions over HTTPS
- **Auth between Supabase and workstation:** API key in request header, validated server-side
- **Monitoring:** Basic Prometheus + Grafana, or simpler: structured logs to stdout + log shipping to a hosted service

### Networking concerns

**The workstation needs to be reachable from Supabase edge functions** for use cases that aren't manual-query-only. Options:

1. **Public IP + Cloudflare Tunnel** — exposes the workstation through Cloudflare's network without opening firewall ports. Free tier sufficient. Recommended.
2. **Tailscale Funnel** — similar idea, simpler setup, requires Tailscale ecosystem.
3. **Direct port forwarding** — works but exposes home/office network. Not recommended.
4. **Run a small VPS as a proxy** — workstation calls home to VPS, VPS exposes the API. Adds latency but cleanest from a security perspective.

The Corsair workstation comes with 2.5G Ethernet and Wi-Fi 6E — both adequate for AI traffic.

### Cost-benefit at Rolliworks scale

At ~17 requests/day with maybe 3-5 AI assistant queries per request × 2-3 team members ≈ **~100-200 AI queries/day**.

- **Cost via cloud APIs (Claude, OpenAI):** ~$0.50-5/day depending on tokens = **$15-150/month**
- **Cost via local workstation:** electricity only, ~$1-3/month at typical use

Payback period on a $2,000-2,300 workstation: 12-24 months at minimum cloud usage, 18-36 months at heavier usage. **Not amazing pure ROI** — the real value is privacy (client data stays local) and capability (large models that wouldn't fit on cloud free tiers, multimodal in a single integrated system, no rate limits).

---

## 5. Roadmap status

### Phase 1 (in progress, May 12 — May 18)
**Goal:** Replace Wix chat. RC handles new-lead intake, message threads, email round-trips bidirectionally.

| Night | Status | Feature |
|---|---|---|
| Mon May 12 | In progress | Atomic-swap Wix→RS→RC intake routing + dual-auth + auto-reply |
| Tue May 13 | Not started | Pattern Y full-content outbound emails |
| Wed May 14 | Not started | Postmark Inbound + DNS + HMAC tokenization (3hr night) |
| Thu May 15 | Not started | Inbound text + photo attachments → RC conversations |
| Fri May 16 | Not started | Embedded photos in saved replies |

**Tuesday morning validation routine** captured in separate `Tuesday_Morning_Validation.md` file.

### Phase 2 (week 3+, exact dates TBD)
- Live shop floor map in portal (replaces placeholder)
- Vertical pre-shop-floor tracker with live RS data
- Multi-item folder/tab view
- Parts approval routing through portal (URL change)
- Intake/inspection photos display with lazy load
- Inspection notes downloadable
- RS booking link + Request photos buttons in composer
- Global portal control panel (on/off toggles)
- Status badge in portal header (small punt from Phase 1 Friday)
- Search results page (already drafted)
- Mobile responsiveness pass

### Phase 3 (later)
- Custom domains (`crm.rolliworks.com`, `my.rolliworks.com`)
- Wix chat retirement (7 days parallel, then cold kill)
- Archival logic (30-day post-completion)
- Team training

### AI Workstation integration (parallel track)
- Phase A: Set up workstation, install Ollama, expose endpoint securely
- Phase B: Build job-lookup AI assistant UI (chat interface in CRM)
- Phase C: Integrate quote-stripping into RC inbound email pipeline
- Phase D: Bracelet ID from intake photos (RW)
- Phase E: Estimate prep assistance (RS)

---

## 6. Decision history — important context

These decisions shaped the architecture and should not be re-litigated unless context has genuinely changed:

**Architectural decisions locked:**
- Wix → RS → RC (not Wix → RC direct). RS owns customer identity.
- Conversations are 1:many with jobs. Data model supports multi-item, UI ships single-job in Phase 1.
- All NEW endpoints authenticated with `x-api-key`. No URL obscurity.
- RC pulls from RS and RW (no push from them to RC for status), with three exceptions: `rs-intake-from-rc` (push from RS at intake), `rs-customer-promoted` (push from RS at conversion), `rs-job-created` (push from RS at Stage 3).
- Server-side enum normalization (RW maps to RC's display enum, not vice versa).
- Pre-existing risks (RS/RW open inbound endpoints, Wix duplicate-submission) are tracked but NOT Phase 1 scope.
- Inbound email provider: Postmark Inbound (decided after RC's recommendation).
- HMAC reply-to tokenization, never guess at conversation matching.
- `unmatched_inbound` table for routing failures + scheduled email alert.
- Hash-fingerprint outbound attachments for inbound dedup (not quote-stripping).
- Wix retirement: 7 days parallel, then cold kill widget. No conversation history migration.

**Strategy decisions:**
- Carrot + stick model: portal becomes more valuable than other channels (carrot), team redirects clients to portal consistently (stick).
- Phase 1 ships functional parity with Wix + one differentiator (embedded saved-reply photos). Status badge defers to Phase 2.
- RW's `ClientReplies` / `NoReplyInbox` stays untouched in Phase 1. Unified communication is Phase 2+.
- Photos are a major Phase 2 carrot. Intake photos, inspection photos, AI-identified watch model.
- Don't replace Gemini OCR with local Gemma 4. Keep specialized tools for specialized jobs.

**Estimation calibration:**
- Claude's time estimates historically run 30-50% too high. Trust closer to half of Claude's first estimate.
- Don't suggest "good stopping points" prematurely. If user says they have 10 hours, plan for 10 hours.

---

## 7. Working with this stack

### Three Lovable AIs as planning partners

Each Lovable AI sees only its own codebase. When planning cross-system features:
1. Send the same orientation context to each AI separately
2. Ask each one for opinions on the parts relevant to its system
3. Bring all three responses together
4. Synthesize, identify mismatches, push back on the AI that has the wrong picture
5. Loop until aligned

The May 12 planning round produced an integration contract (v2) plus follow-ups via this pattern. **Three rounds of cross-AI discussion is usually enough.** A fourth round produces invented concerns more than real ones.

### Building features

For each feature:
1. Scope the feature with Claude (this chat) — what it does, why, what gets in/out
2. Ask Lovable AI for code-reality check — what existing code does it touch, concerns, etc.
3. Synthesize, write the build prompt
4. Paste the build prompt into the relevant Lovable chat
5. Verify the result against acceptance criteria

For cross-system features, draft separate build prompts for each system and coordinate deploy order. May 12's atomic-swap pattern (RS deploys first, then RC) is a good template.

### When something feels off

If the conversation feels like it's starting over from scratch, check:
- Are there saved planning docs from earlier sessions? (Most likely)
- Is the user remembering decisions made in a different chat instance? (Common — Claude doesn't see across chat history)
- Has the integration contract or this handoff doc drifted from reality?

Recovery prompt for old chats lives in `RC_Recovery_Prompt.md` (in earlier session outputs).

---

## 8. What's NOT in this handoff

- Specific UI design details (covered elsewhere)
- Database schemas in full (consult Lovable AIs for current state)
- Exact secret values (consult Lovable Cloud settings)
- Marketing copy or client-facing tone guidelines
- Pricing for AI cloud services (consult vendor pricing pages)

If a new chat needs these, ask the user or the relevant Lovable AI directly.

---

## 9. Quick reference — file index from prior sessions

These files were produced in earlier sessions and should be in user's local storage:

- `RolliConnect_Saved_Replies_Prompt.md` — initial saved replies build
- `RolliConnect_Saved_Reply_Card_Label_Fix.md` — UI tweak
- `RolliConnect_Global_Search_Prompt.md` — search feature (already built)
- `RC_Recovery_Prompt.md` — recovery prompt for old chats
- `RC_Lovable_Orientation_Prompt.md` — Round 1 planning prompt to RC AI
- `RC_Lovable_Contract_Followup.md` — Round 2 contract draft request
- `Three_AI_Briefing_Prompts.md` — Round 3 cross-AI orientation
- `Three_AI_Followup_Prompts.md` — Round 3 follow-ups
- `Monday_Build_Prompts.md` — May 12 atomic-swap build prompts (RS + RC)
- `Tuesday_Morning_Validation.md` — Day-after validation routine

---

## 10. How to use this doc

**For a fresh Claude chat:** read sections 1-3 to get context, then jump to whatever the user wants to work on. Reference sections 4-9 as needed.

**For the user:** keep this doc updated as decisions get made. When something significant changes (new architectural decision, new system added, phase complete), update the relevant section.

**When this doc and a Lovable AI disagree:** Lovable AI is closer to the actual code. Trust Lovable for current implementation, trust this doc for intent. If they really conflict, that's a sign the code drifted from intent — worth investigating.
