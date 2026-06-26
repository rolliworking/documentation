# Rolliworks — Future Integration List

**Purpose:** Capture every system, module, or tool that will *eventually* need to send data to or read data from RS / RW / RC, even if it doesn't exist yet or is still fuzzy.

**One-line goal anchor:** _[paste the rebuild one-line goal here]_

---

## Known integrations

### 1. RolliTime (watch testing program)
**Status:** Spec'd. See `integrations/ROLLITIME-INTEGRATION.md` for full contract.
**Language:** Python (FastAPI) per D-017.
**Summary:** Owns testing data + dial-photo library. RS is source of truth for caliber/reference data. RolliTime pushes status to RW, exposes test results + photos to RS.
**Reserved schema slots:** `shared.calibers`, `shared.reference_models`, `rollitime.*` (own DB per contract).

---

### 2. M3KE chatbot embedded in RS
**Status:** M3KE v1 live as escalation tool. Chatbot-in-RS is next phase.
**Language:** Python per D-017.
**Produces:** Embeddings, classifications, interaction logs, escalations.
**Reads:** RS data (currently JWT-blocked).
**Reserved schema slot:** `ai_modules.m3ke_*`

---

### 3. Jarvis
**Status:** Early / fuzzy.
**Language:** Python per D-017.
**Produces:** TBD.
**Reads:** TBD.
**Reserved schema slot:** `ai_modules.jarvis_*` (empty placeholder).

---

### 4. Watch Tools Knowledge Module
**Status:** Early / fuzzy.
**Produces:** TBD.
**Reads:** TBD — likely caliber/reference data from RS.
**Reserved schema slot:** `ai_modules.watch_tools_*`

---

### 5. Authenticator App
**Status:** Future. See `integrations/ROLLITIME-INTEGRATION.md` §5.
**Language:** Python per D-017.
**Reads:** RolliTime's dial-photo library (consumer, read-only).
**Reserved schema slot:** None in shared DB (consumes RolliTime's API).

---

### 6. RolliCurator (knowledge harvest wizard)
**Status:** Concept stage. See `concepts/ROLLICURATOR-CONCEPT.md`. Decision: D-012.
**Language:** Python per D-017.
**Summary:** Wizard module that pre-scrubs multiple AI sources, presents multiple-choice quizzes to humans, captures validated expertise into master tables.
**Produces:** Validated entries on master tables; audit trails per contributor; high-value proprietary dataset.
**Reads:** All master tables (`shared.*`, `rs.*` reference data); plus external AI APIs (ChatGPT, Opus, Perplexity, Gemini).
**Reserved schema slots:** `ai_modules.curator_*` for audit trails, contributor records, question history. Master tables themselves require `status`, `validated_by`, `validated_at`, `confidence` columns (part of D-011 implementation).

---

### _(add new entries here as they surface)_

---

## Cross-cutting questions to answer once

**Q1. How will AI modules authenticate to read RS / RW / RC data?**
> Tracked in `REBUILD-PREREQUISITES.md` #3.

**Q2. Where do AI modules write their outputs?**
> `ai_modules.*` schema. Each module gets its own table set.

**Q3. How does data from external programs authenticate to write into the shared DB?**
> Tracked in `REBUILD-PREREQUISITES.md` #3 + #4.

**Q4. Capture-now vs. defer policy for fuzzy future data?**
> Capture if cheap (JSONB column). Defer if it requires a new table. Revisit quarterly.

**Q5. Reference data — who owns it across the suite?**
> **RolliSuite is source of truth.** RW, RolliTime, and all future modules read from RS-authoritative shared tables. Tracked in `REBUILD-PREREQUISITES.md` #2 and D-002.

**Q6. Who reviews this list and decides what gets a reserved schema slot?**
> Michael + whoever leads the rebuild, together, once per quarter.

**Q7. How are living master tables maintained over time?**
> Through RolliCurator (see #6 above and D-011, D-012).

---

_End of list. Revisit quarterly._
