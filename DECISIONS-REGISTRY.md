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

## How to add a new decision

When a decision gets made in a session, append a new entry here. Use the next D-### number. Include date, decision, context, status, and source chat. If the decision overrides an earlier one, mark the earlier one as "Superseded" and link both ways.

---

_End of registry. Append-only. Never delete._
