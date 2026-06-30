# Rolliworks — Decisions Registry

**Purpose:** Every architectural, tooling, and process decision made across all planning sessions. Append-only. Never delete entries — if a decision is reversed, add a new entry referencing the old one.

**One-line goal anchor:** _[paste the rebuild one-line goal here]_

**Format per entry:**
- **Date** — when decided
- **Decision** — what was decided, in one line
- **Context** — what problem it solved, in 1-3 sentences
- **Status** — Active / Superseded / Pending implementation
- **Supersedes** — links to prior entries if applicable
- **Source** — which chat/session

---

## D-001 — Path A: PartsWiki standalone first
**Date:** 2026-05 (prior planning sessions)
**Decision:** Build PartsWiki as a standalone app before deciding on a full rebuild of RS / RW / RC. Defer the full rebuild decision 3 months until evidence exists.
**Context:** Avoids rebuilding three live production apps from incomplete documentation. PartsWiki is unblocked by RepairShopr access, not by the existing apps' internals.
**Status:** Active
**Source:** HANDOFF.md, PLANNING-QUESTIONS.md

---

## D-002 — RS is source of truth for caliber and reference-model data
**Date:** 2026-06-07
**Decision:** RolliSuite (RS) is the authoritative source for caliber data, reference-model data, ref# → caliber mappings, caliber specs, and minimum tolerances. RW, RolliTime, and all future modules read from RS-authoritative shared tables.
**Context:** Today RS and RW each maintain their own reference libraries and have drifted. The rebuild consolidates these into a single shared schema with RS owning writes.
**Status:** Active (pending implementation — see REBUILD-PREREQUISITES.md #2)
**Source:** Session 2026-06-07

---

## D-003 — Shared Supabase, separate apps
**Date:** 2026-06-07
**Decision:** RS, RW, RC will share a single Supabase project with separate schemas (`rs.*`, `rw.*`, `rc.*`, `shared.*`, `ai_modules.*`). Apps remain separate codebases with separate deployments.
**Context:** Solves cross-app data drift. Enables AI modules to read from one DB instead of three. Apps stay independently deployable.
**Status:** Active (pending implementation)
**Source:** Session 2026-06-07

---

## D-004 — Local hardware roles narrowed
**Date:** 2026-06-07
**Decision:** Corsair 96GB workstation is the Ollama server. Qwen handles documentation drafting. gbrain (local MCP) acts as persistent librarian. Local hardware is NOT used for code generation or adversarial review of Opus.
**Context:** Operator (Michael, non-developer) cannot verify code-level output. Local hardware is best at high-volume reading tasks.
**Status:** Active
**Source:** Session 2026-06-07

---

## D-005 — Tool stack for rebuild pipeline
**Date:** 2026-06-07
**Decision:** Defined eight-tool pipeline: Existing apps + SQL + docs repo (source of truth), Lovable (memory helper), Local Ollama/Qwen (refinery), gbrain (librarian), Opus (planner), Cursor interactive (builder), Cursor Cloud Agents (bounded maintenance), Emergent (module rebuilder).
**Context:** Defines clear role per tool, prevents overlap and drift.
**Status:** Active
**Source:** Session 2026-06-07

---

## D-006 — Ruflo not adopted
**Date:** 2026-06-07
**Decision:** Multi-agent orchestration tools (Ruflo) are not adopted.
**Context:** Solves parallelism problems we don't have. Amplifies drift because we lack architect-level review capacity.
**Status:** Active
**Source:** Session 2026-06-07

---

## D-007 — gstack adopted selectively
**Date:** 2026-06-07
**Decision:** Adopt gstack's planning skills (`/office-hours`, `/plan-ceo-review`, `/retro`, `/spec`) and review skills (`/review`, `/qa`, `/document-release`). Skip iOS, multi-agent, parallel sprint skills.
**Context:** Provides structured prompts without requiring developer-level operation.
**Status:** Pending installation
**Source:** Session 2026-06-07

---

## D-008 — RolliTime integration contract accepted
**Date:** 2026-06-07
**Decision:** RolliTime is a separate system with its own database. Integrates with RS, RW, and the authenticator app via API contracts. Owns testing data and the dial-photo library.
**Context:** RolliTime needs to be designed and built; this contract defines its boundary.
**Status:** Active
**Source:** Session 2026-06-07. Full spec: `integrations/ROLLITIME-INTEGRATION.md`

---

## D-009 — Response format: shorter, fewer questions
**Date:** 2026-06-07
**Decision:** Claude responses are kept short. Maximum 1-2 questions per turn.
**Context:** Long responses with many questions were creating decision backlog.
**Status:** Active
**Source:** Session 2026-06-07

---

## D-010 — Canonical ecosystem data model
**Date:** 2026-06-07
**Decision:** Rolliworks operates as a single watch service center ecosystem with one canonical data model. Every entity (customer, watch, reference, caliber, bracelet, dial, part, service) is defined once in a shared master table. Every app (RS, RW, RC, RolliTime, M3KE, Jarvis, future modules) reads from and writes back to these masters. Naming is consistent across code, schema, docs, and conversation.
**Context:** Today, each app invented its own naming and tables for the same concepts (e.g. caliber drift between RS and RW). This is the umbrella principle that drives D-002, D-003, D-011, and the glossary requirement. A buyer doing due diligence on a watch services platform values one canonical model far above scattered duplicate systems.
**Status:** Active (foundational principle, drives implementation in other decisions)
**Source:** Session 2026-06-07

---

## D-011 — Living masters with promotion workflows
**Date:** 2026-06-07
**Decision:** Master tables in the canonical model are *living* — they grow and improve as the ecosystem operates. Every app contributes to them with appropriate permissions. Rows can exist in states: `stub`, `partial`, `complete`, `validated`, `authoritative`. Each master has a defined owner and a promotion workflow. Audit trails track all contributions.
**Context:** Current "39 incomplete stubs" problem in calibers is the predictable result of having no promotion workflow. Without explicit ownership and review cadence, masters accumulate junk. Owners per master: customers→Vianna, references/calibers/bracelets→Watchmaker supervisor, dials→Appraisal lead, parts→M3KE-integrated workflow.
**Status:** Active (pending implementation alongside rebuild)
**Source:** Session 2026-06-07

---

## D-012 — Curated knowledge harvest module (RolliCurator — working name)
**Date:** 2026-06-07
**Decision:** Master table completion runs through a dedicated wizard module:
1. System detects gaps (stubs, missing fields, conflicts)
2. Pre-scrubs multiple AI sources (ChatGPT, Opus, Perplexity, Gemini) + curated watch references in the background
3. Auto-fills high-confidence entries; escalates ambiguous ones
4. Presents ambiguous entries as numbered multiple-choice quizzes ("0" for other-with-text-entry)
5. Supports multiple contributors with consensus-based confidence scoring
6. Optimized for numeric keypad input — rapid bench-side data entry
7. Maintains audit trails per contributor and per AI source

The resulting validated dataset is a strategic acquisition asset — differentiated from any public AI's training data by being expert-validated and operationally proven. Eventually becomes training data for proprietary Rolliworks LLMs.
**Context:** Solves D-011's promotion workflow problem with engagement. Open-ended questions have too much friction; multiple-choice from pre-scrubbed data is 10x faster. "None of these" answers capture proprietary expertise.
**Status:** Concept stage. Full spec: `concepts/ROLLICURATOR-CONCEPT.md`
**Source:** Session 2026-06-07

---

## D-014 — Phase 1 agent skills in rebuild-hq
**Date:** 2026-06-21
**Decision:** Adopt five Phase 1 agent skills in `rebuild-hq/.claude/skills/`, plus `CLAUDE.md` (doctrine) and `.cursor/rules/` (Cursor parity). The rebuild-hq repo holds agent operating procedures; this documentation repo holds all decision records and domain knowledge.
**Context:** Rebuild work spans multiple app repos and requires repeatable procedures for mapping legacy behavior, designing canonical schema, generating bounded task packets, reviewing PRs, and orchestrating cutovers. Without codified skills, each session reinvents process and scope creeps. Phase 1 skills close the loop from D-010 and D-011 into executable, reviewable work.
**Status:** Active
**Skills:** `mapping-legacy-workflows`, `modeling-canonical-data`, `generating-task-packets`, `reviewing-prs-governance`, `orchestrating-cutover-tests`
**Companion files (in rebuild-hq):** `CLAUDE.md`, `.cursor/rules/rebuild-hq-doctrine.mdc`
**Repo boundaries:** `rolliworking/rebuild-hq` = skills + doctrine; `rolliworking/documentation` = domain docs + all decisions; app repos = code on `test` branch
**Source:** rebuild-hq kickoff session 2026-06-21

---

## D-015 — Chain-of-custody surveillance integration
**Date:** 2026-06-25
**Decision:** The rebuild includes a chain-of-custody surveillance layer. IP cameras integrate with the suite at the safe and at incoming-package stations. Every customer label scan logs to the security camera system, tagged with station ID and timestamp. Each scan is queryable: from a label scan, retrieve the corresponding video footage. A bulk safe-audit capability allows scanning every item in storage to confirm inventory.
**Context:** Per Mike's final reflection, the single biggest operational risk in the shop today is "not knowing which job which tech has and for how long. Not knowing what's waiting in the safe. No trail or log for chain of custody within the shop." This is not a software bug — it's a missing system. Theft deterrence and accountability both require it. Solves multiple downstream problems: orphan-component detection, components-awaiting-other-components visibility, staff accountability.
**Status:** Active (concept stage — implementation deferred to after core rebuild)
**Scope:** Hardware: IP cameras at safe + each intake/pickup station. Software: scan-log → camera-frame query. Operational: bulk safe-audit workflow. Integration: tie to existing label scan flows in RS and pickup flows.
**Dependencies:** Camera system selection (Nest, Reolink, dedicated NVR, etc.). Label-scan logging infrastructure (must exist in shared schema). Auth model for accessing footage (who can query, who can view).
**Source:** Session 2026-06-25 — Mike's Final Reflection

---

## D-016 — QR-driven watchmaker bench workflow
**Date:** 2026-06-25
**Decision:** The watchmaker bench workflow is rebuilt around QR code scanning, not menu navigation. Each watchmaker has a small set of process QR codes physically present at their bench (In Testing, Downgrade, Parts Request — plus any others determined necessary). Workflow: scan watch ref-serial → scan process QR → action executes. Optimized for iPad gestures and fast bench work. Voice-to-text supports job notes for hands-busy contexts.
**Context:** Today's RW uses a drop-down menu for status changes on iPad. The iPad UI is "basically a web view, not streamlined for iPad gestures." Per Mike: "scan 1601-743849 then process QR. Fast and easy." Reduces taps from ~5-7 to 2. Job notes and comments are currently missing from RW; voice-to-text + all-staff contribution closes the knowledge gap.
**Status:** Active (depends on RolliWorking rebuild)
**Scope:** Generate process QR codes (printed, physical at bench). Build scanner-first iPad UI for RW. Add job notes/comments table to shared schema. Voice-to-text integration (iOS native or web API).
**Dependencies:** RolliWorking rebuild (the bench UI surface). Notes/comments writable by all RW users (auth + RLS). iPad camera-as-scanner working reliably (or external Bluetooth scanner).
**Related:** D-008 (RolliTime integration — RW pulls testing data, related to In-Testing status). Wishlist W29–W30 (QR bench workflow, voice-to-text job notes).
**Source:** Session 2026-06-25 — Watchmaker bench iPad observations

---

## D-017 — Hybrid language strategy (TypeScript + Python)
**Date:** 2026-06-25
**Decision:** The Rolliworks ecosystem uses a hybrid language strategy. Customer-facing operational apps stay in TypeScript / React. Data, ML, AI, and computer-vision modules are built in Python (FastAPI).

**Language assignments:**

| App / Module | Language | Rationale |
|---|---|---|
| RolliSuite (RS) | TypeScript | Most complex existing codebase; integration-heavy; high rewrite risk; user-facing |
| RolliWorking (RW) | TypeScript | iPad bench UI; tightly coupled to RS workflows |
| RolliConnect (RC) | TypeScript | Customer-facing portal; web-first |
| RolliTime (RT) | Python (FastAPI) | Data-heavy testing, OCR pipelines, dial-photo library mgmt, future ML training data source |
| M3KE | Python | LLM / NLP work; already partially Python |
| Authenticator | Python | Computer vision, ML training, image processing |
| Jarvis | Python | LLM agent infrastructure |
| RolliCurator | Python | Multi-AI orchestration, embeddings, consensus engine |
| Shared Supabase (PostgreSQL) | Language-agnostic | All apps and modules connect via SDK / REST |

**Context:** Two paths were considered: (a) keep everything in TypeScript matching the current Lovable stack, or (b) rebuild RS/RW/RC in Python for PE valuation upside. The hybrid path captures the Python advantages where they actually matter (ML, CV, data engineering) without the rewrite tax on three working operational apps. Strategic acquirer path (Richemont, Bucherer, watch industry consolidators) is more likely than PE — those buyers value operational integration over language choice, lowering the PE-premium argument. M3KE, Authenticator, Jarvis, and RolliCurator are all greenfield builds where Python is the right call regardless.

**Architectural implications:**
- All apps communicate via shared PostgreSQL (Supabase) — language boundary is at the database, not in app-to-app calls
- Python modules expose REST/FastAPI endpoints when other modules need to consume their outputs
- AI module schema (`ai_modules.*`) is the contract surface for Python modules
- Authentication and RLS patterns work identically across both stacks (Supabase auth supports both SDKs)
- Cross-app contracts (RolliTime ↔ RS, etc.) become language-agnostic API contracts, not shared-codebase imports

**Status:** Active (foundational language strategy; informs all future build slices)

**What this enables:**
- Cleaner ML and CV integration for Authenticator and future authenticity scoring
- Python ecosystem access for data engineering work (dial-photo library, training datasets)
- Easier hiring for ML/data roles (Head of Applied AI, etc.)
- Aligned with Rollie Group's AI-augmented operations direction

**What this avoids:**
- Multi-month rewrite of three working production apps (RS/RW/RC) into a new language
- Operational risk of three simultaneous cutovers
- Cursor + Opus context-window stress of redoing TypeScript work in Python
- Loss of existing test, deployment, and Lovable-compatible infrastructure

**Companion files to update:**
- `CLAUDE.md` — add language-split note to architectural section
- `GLOSSARY.md` — add Python/FastAPI as recognized stack alongside TS/React
- `BUILD-PROCESS.md` — note that SPEC stage must identify target language for the slice
- `FUTURE-INTEGRATIONS.md` — confirm Python module entries (M3KE, Jarvis, etc.) reflect the language

**Open follow-ups (not decisions yet):**
- Python module deployment target — Emergent supports Python, but specific hosting (Fly.io, Railway, separate) is TBD per module
- Python skill files — when Phase 2/3 trigger fires, draft equivalents to the existing TS-focused skills (e.g., `modeling-canonical-data` is already language-agnostic; `reviewing-prs-governance` needs a Python-specific check section)

**Source:** Session 2026-06-25 — strategic upside analysis + hybrid recommendation

---

## D-018 — Question snapshot discipline (amended 2026-06-28)
**Date:** 2026-06-25 (initial), amended 2026-06-28
**Decision:** Questions accumulated in `HUMAN-QUEUE.md` (and the "Open questions" sections of discovery files) are periodically exported to dated Word documents. Snapshots live in Dropbox at `G:\Dropbox\__AI\emergent-hq-human-queue\` to enable cross-device review and answering. The markdown source of truth (HUMAN-QUEUE.md, HUMAN-ANSWERS.md) stays in git.

**Filename pattern:** `QUESTIONS-YYYY-MM-DD.docx`

**Cadence — generate a new snapshot when any of these fire:**
- Every 72 hours (alongside the build digest per BUILD-DIGEST-TEMPLATE.md)
- When HUMAN-QUEUE.md has 10+ open questions
- Before any Opus planning session
- Before any Qwen autonomous run lasting 4+ hours

**Context:** Markdown question queues are hard to scan on mobile, hard to print, and don't enforce a review cadence. Word docs with answer-box formatting solve all three: scannable offline, printable, structured for human input. Saving to Dropbox (not the git repo) auto-syncs snapshots across all devices — Legion, work computer, iPad, phone — without manual copying. Answered docs return via the same Dropbox folder.

**Why Dropbox, not the git repo:**
- Word docs are binary; git history adds no value
- Cross-device access without git pull/push overhead
- Dropbox auto-sync removes manual save/copy steps
- HUMAN-QUEUE.md and HUMAN-ANSWERS.md (the actual source of truth) remain in git

**Roles:**
- **Qwen / Cursor:** Logs questions to HUMAN-QUEUE.md in the format specified in QWEN.md
- **Opus / Cursor:** Generates the Word doc snapshot on cadence, saves to Dropbox
- **Michael:** Opens the Word doc on any device (Dropbox auto-syncs it), answers questions during downtime
- **Cursor:** Reads answered Word doc from Dropbox, updates HUMAN-ANSWERS.md in the repo, commits and pushes
- **Qwen:** Reads HUMAN-ANSWERS.md at session start to unblock work

**Folder structure:**
- `G:\Dropbox\__AI\emergent-hq-human-queue\` — root folder for snapshots
- `G:\Dropbox\__AI\emergent-hq-human-queue\QUESTIONS-YYYY-MM-DD.docx` — each snapshot
- `G:\Dropbox\__AI\emergent-hq-human-queue\answered\` — optional subfolder; move answered docs here once their answers have been extracted to HUMAN-ANSWERS.md (preserves historical record)
- `documentation/queue-exports/README.md` (in repo) — redirect document pointing to the Dropbox location

**Status:** Active
**Supersedes:** Original D-018 (which placed snapshots in `documentation/queue-exports/` inside the repo)
**Source:** Session 2026-06-28 amendment

---

## D-019 — Two-stage intake pattern
**Date:** 2026-06-28
**Decision:** Intake is a two-stage process. Stage 1 captures possession; Stage 2 verifies and commits to inventory. The two stages are tracked separately and must be auditable against each other.

**Stage 1 — Possession layer:**
- Drop-off page (in-person intake) + Shipping Receive page (package arrivals) are the source of truth for "what physically arrived"
- Captures: client, tracking number (if shipped), arrival photos, condition notes, who received it, when
- Writes to: possession tables (rs.intake_possession or equivalent)
- Status: "received" — meaning the item is in the building, but not yet committed to inventory

**Stage 2 — Verification + inventory commit:**
- Receive Watch page is a verification gate, not just a UI step
- Verifies Stage 1 data matches physical reality (component count, condition, serial numbers)
- On verification: commits the asset to inventory (writes to client_property or equivalent)
- On disagreement: logs discrepancy to audit log, holds stage transition
- Status: "verified" / "in_inventory" — meaning the item has cleared verification

**Audit requirement:**
- Every Stage 1 → Stage 2 transition writes to an audit log
- Discrepancies (Stage 1 said X, Stage 2 confirmed Y) get explicit log entries with staff attribution
- The audit log is searchable by client, asset, staff, date, and status
- Foundation for the chain-of-custody system (D-015)

**Context:** The current code conflates Stage 1 and Stage 2 — intake writes directly to client_property without an explicit verification gate. This is the upstream root cause of Shop Floor's "components aren't registered" failure and the absence of any theft-prevention audit trail. Michael's #1 operational risk per workflow capture (chain of custody / theft prevention) maps directly to this pattern.

**Architectural implications:**
- Schema must support both stages with explicit linking
- A possession-stage record without a verification-stage record means an asset is "in the building but unverified"
- A verification-stage record must reference its possession-stage record (FK)
- Audit log table must exist as a shared.audit_log surface for cross-app querying
- This is the foundation for the visual asset inventory layer (W-33) and the audit log (W-34)

**Status:** Active (foundational architectural pattern; affects multiple rebuild slices)

**What this enables:**
- Theft prevention via auditable two-stage commit
- Disagreement detection (when verification finds Stage 1 inaccuracies)
- Multi-item intake with per-item verification
- The visual asset layer per W-33 (every asset has a clear stage status)

**What this avoids:**
- Silent inventory writes without verification
- The current "components aren't registered" Shop Floor failure root cause
- Loss of accountability when assets move between staff

**Companion files to update:**
- `PLANNED-CODEBASE.md` — note the two-stage intake pattern in the RolliSuite operational section
- `WISHLIST.md` — W-33 and W-34 (just added)
- Future SPEC for intake rebuild — first SPEC under BUILD-PROCESS must reflect this

**Source:** Session 2026-06-28, A-20260628-003 (Michael's answer to Q-20260626-003)

---

## D-020 — Silent failure is a banned pattern; Receive Watch is component source of truth

**Date:** 2026-06-28
**Decision:** Two related architectural commitments emerging from QBO investigation and intake workflow answers:

**Part A — Silent failure is banned.** Every async job, cron, webhook, edge function, and background task in the rebuild MUST produce observable success/failure status. Writes that return HTTP 200 while the durable record is silently empty are unacceptable. Specifically prohibited patterns:
- INSERT failures that proceed to UPDATE-with-zero-rows-affected without erroring
- Cron functions that log "started" but never reach "success" or "failed"
- Webhook handlers that fail-open when configuration is missing
- Schema drift between code writers and table constraints

Every long-running job must have: visible monitoring, explicit timeout handling, idempotent operations, chunked processing where dataset size warrants it, and durable audit trail entries written BEFORE the operation begins (not after).

**Part B — Receive Watch is the source of truth for component creation.** Component records (whatever the rebuild calls them — `job_components`, `intake_components`, etc.) must be created at the Receive Watch verification stage (Stage 2 of D-019's two-stage intake pattern). They must NOT be derived from estimate markers, item types, or upstream RW intake hooks. The operator selecting components at Receive Watch is the canonical creation event.

**Context:** Lovable AI's audit of the QBO integration confirmed a recurring "silent failure" pattern: code writes return success while durable records are missing. The customer sync cron has been broken since 2026-01-12 (162 stuck rows). The qbo_sync_log CHECK constraint silently rejects invoice rows. The `finished_at` column phantom write silently fails. This is the same pattern previously found in inspection-supersession and estimate-immutability investigations.

Combined with Michael's confirmation that estimate markers are merely "suggestions" and Receive Watch backfills the classification, the rebuild architecture must enforce: (a) writes that mean nothing on success must be eliminated, (b) component data of record is created at human verification time, not derived from upstream hints.

**Architectural implications:**
- Every cron in the rebuild has a `runs` table tracking start/end/status/payload
- Every webhook handler logs the raw payload BEFORE dispatch
- Every CHECK constraint must match what the codebase writes (or vice versa)
- The Shop Floor data model uses station_id as a first-class authoritative field, not derived from status+department heuristics
- The Receive Watch page UI must include component creation controls with department selection
- The rollisuite-intake edge function (legacy boundary between RS and RW) becomes a sync/notification mechanism, not a creator

**Status:** Active (foundational pattern; affects every async/durable-write path in the rebuild)

**Companion items:**
- Q-005 answer (A-20260628-010) — customer cron silent failure evidence
- Q-005-B answer (A-20260628-011) — qbo_sync_log schema drift evidence
- Q-004-1 answer (A-20260628-006) — estimate markers are suggestions
- Q-004-3 answer (A-20260628-008) — Receive Watch as component creator
- W-37 — Shop Floor drag-and-drop GUI (depends on station_id being authoritative)
- PROD-FIX-001 — patches the qbo_sync_log drift in the current Lovable app

**Source:** Session 2026-06-28 (PM), Lovable AI audit + Michael's answers

---

## How to add a new decision

When a decision gets made in a session, append a new entry here. Use the next D-### number. Include date, decision, context, status, and source chat. If the decision overrides an earlier one, mark the earlier one as "Superseded" and link both ways.

---

_End of registry. Append-only. Never delete._
