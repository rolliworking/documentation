# RolliCurator — Concept Document

> **Status:** Concept stage. Not yet built.
> **Decision:** D-012 in `DECISIONS-REGISTRY.md`.
> **Working name:** RolliCurator (alternatives considered: RolliMaster, RolliWiki, RolliBrain).

---

## What it is

A dedicated wizard module that turns master table completion into an engaging, fast, multi-contributor activity. Sits alongside RC / RW / RS / RolliTime as the **ecosystem maintenance layer**.

Its job: keep the canonical data model (D-010) clean, complete, and accurate as the business operates.

---

## Why it exists

The living-masters model (D-011) creates a maintenance burden: stub rows pile up if nobody curates them. Today's RS+RW system has 39 incomplete caliber stubs as evidence of this exact failure.

Asking humans to "go fill in the spreadsheet" doesn't work — too tedious, too low priority.

RolliCurator solves that by:
- Doing the research *for* the human (AI scrubs the web first)
- Asking *short, specific* questions (multiple choice with numeric input)
- Making it *fun* (gamification, leaderboards opt-in, visible impact)
- Letting *multiple humans* contribute (consensus validation)

---

## Core mechanics

### 1. Gap detection

Continuously scans master tables for:
- Stub rows (`is_complete = false`)
- Missing required fields on otherwise-complete rows
- Conflicting values across related tables (e.g. caliber drift between `calibers.caliber_number` and `model_references.caliber`)
- Stale entries (last validated > N months ago)

Each gap becomes a candidate question.

### 2. Pre-scrub (background)

For each candidate question, the system queries multiple sources in parallel:
- ChatGPT / Claude Opus / Perplexity / Gemini API endpoints
- Curated watch reference sites (manufacturer archives where accessible, expert databases)
- Internal cross-references (does adjacent reference data suggest an answer?)

Results are consolidated:
- **High consensus** → auto-fill candidate, mark as "AI-confirmed, awaiting human spot-check"
- **Partial consensus** → multiple-choice question with most-likely options
- **No consensus / no data** → open question for human

This step takes 10-30 seconds per question, runs in the background. Humans never wait.

### 3. Question presentation

For questions escalated to humans, the wizard presents:

```
Which caliber(s) ship in the Rolex 1601 Datejust?

AI research summary: 1570 most cited, 1575 second-most, 1560 mentioned as rare variant

1. 1570
2. 1575
3. 1560
4. 1530
0. Other (type answer)

Press number(s) and Enter, or 0 to type:
_
```

Mechanics:
- **Numbered options** for rapid keypad input
- **"0" for other** with free-text fallback
- **Multi-select** by typing multiple digits (e.g. `12` for 1570+1575)
- **Skip** key for "come back to this later"
- **Back** key for "go to previous question"
- **AI summary** visible so the human sees what was found

### 4. Consensus & confidence

Multiple humans may answer the same question over time. The system tracks:

- **Single-validator answers** — confidence 60%, marked "needs second opinion"
- **Two validators agree** — confidence 90%, marked "validated"
- **Three+ validators agree** — confidence 99%, marked "authoritative"
- **Disagreement** — flagged for senior review with both answers visible

A row's status progresses: `stub` → `drafted` → `confirmed` → `validated` → `authoritative`.

### 5. Audit trail

Every answer captures:
- Who answered (contributor ID)
- When
- What AI sources were presented
- What the human picked (including "other" text)
- Time-to-answer (UX metric, also catches rushed answers)

This makes the dataset *defensible* — every entry has a chain of custody.

---

## Contributor experience

### Roles & question routing

Different contributors see different questions based on expertise:

- **Watchmaker supervisor** — rare references, complex caliber questions, family-vs-movement disambiguation
- **Junior watchmakers** — common references, learning-friendly questions
- **Vianna** — customer data, intake-related questions
- **Appraisal lead** — dial styles, condition rubrics
- **Michael R. Michaels (incoming partner)** — high-end / Richemont-era questions

Routing prevents "wrong person, wrong question" and ensures expertise lands where it belongs.

### Engagement mechanics

Borrowed from platforms that have figured this out (Wikipedia, Duolingo, Stack Overflow):

- **Streaks** — "You've contributed 5 days in a row"
- **Domain expertise** — "You're the top contributor on Submariner data this month"
- **Visible impact** — "Your 47 contributions this week populated reference data used by 23 service requests"
- **Quick sessions** — 10 questions in 3 minutes, designed for between-jobs use
- **Notifications** — "5 quiz questions waiting for you" (opt-in)
- **Leaderboards** — opt-in only; some teams love them, others alienated

### Mobile-first

Watchmakers can do quizzes:
- At the bench between jobs
- During lunch on phone
- On a tablet with a Bluetooth number pad ($15 accessory — significant productivity boost)

Desktop also fully supported for office staff.

---

## Architecture (rough)

Five components, each independently buildable:

1. **Gap Detector** — scans master tables, identifies candidates
2. **Web Scrub Orchestrator** — multi-AI + curated source queries, result consolidation
3. **Question Presenter** — wizard UI, numeric input, mobile/desktop
4. **Consensus Engine** — validator tracking, disagreement flagging, confidence scoring
5. **Write-Back Layer** — updates master tables, maintains audit trails

Each can be built and tested independently. None require the wizard itself to "understand" watch domain — that's outsourced to the AI sources.

---

## Why this is a strategic asset

The validated dataset that accumulates from RolliCurator is *not* the same as what any public AI knows:

- AI's training data is messy, includes errors, often outdated
- RolliCurator dataset is **expert-validated** by working watchmakers
- "None of these" answers capture knowledge that *isn't* in public AI training data
- Audit trail proves authenticity (defensible at sale)

Over months/years, you accumulate a proprietary database of Rolex/Tudor reference knowledge — calibers, dial variants, bracelet eras, parts compatibility, common failure modes, tolerance specs, market notes — that's been validated by the people who actually service these watches.

**This is differentiated, monetizable IP.** A strategic acquirer (Anthropic for training data, Hublot / Richemont / Bucherer for operations, watch-tech startups for licensing) would value this.

Fits the "byproduct asset thesis" — strategic assets accumulate at near-zero marginal cost as the shop operates.

---

## Long-term: training data for proprietary models

The accumulated validated dataset becomes the training set for Rolliworks-specific LLMs:

- M3KE chatbot in RS — answers customer questions with authoritative data
- Jarvis — operations assistant trained on internal expertise
- Authenticator app — trained on dial photos + RolliCurator-validated dial styles
- Future modules — all benefit from the same data layer

This is a long-term play (12+ months out) but worth designing the data structure to support it from day one.

---

## Open design questions

1. **Build vs. integrate** — build from scratch, or layer on existing quiz/survey platforms (Typeform, Google Forms, custom)? Likely custom because of the AI-scrub-first mechanic.
2. **Where does it live** — its own app, or a module inside RC? Probably its own app for clean separation.
3. **Auth** — uses the shared Supabase auth or its own?
4. **Compensation** — is contribution part of paid job duties, or extra (gift cards, time off)? Affects engagement design.
5. **External contributors** — eventually, could we open a curated subset of questions to vetted external watchmakers? Builds the dataset faster but raises validation complexity.

---

## Dependencies

Cannot be built until:
- D-010 / D-011 are implemented (canonical model exists, living masters defined)
- Shared Supabase rebuild is complete (D-003)
- Master table schemas are stable

Therefore: **post-rebuild module.** Build only after the canonical model is real.

But: **design now, defer build.** The canonical model rebuild should anticipate RolliCurator's needs (state machine for promotion workflow, audit trail columns, contributor identity model).

---

## Next steps when RolliCurator becomes active project

1. Write detailed spec from this concept doc
2. Build component 1 (Gap Detector) against the canonical model
3. Build component 2 (Web Scrub Orchestrator) as a standalone service
4. Build component 3 (Question Presenter) as a minimal mobile-first UI
5. Test with watchmaker supervisor on a single master (calibers) for one month
6. Iterate based on real contributor feedback
7. Expand to other masters

---

_End of concept document. Revisit when ready to spec the build._
