# RolliTime — Integration Contract (Living Document)

> **Purpose:** Defines the data that flows **into** and **out of** RolliTime, and the
> shared contracts other systems build against. This is separate from the internal build
> spec (`SPEC.md`) on purpose — it is the **shared reference** for the upcoming Emergent
> rebuild of the ERP suite (RolliSuite / RolliWorking) and for the authenticator app.
>
> **Status:** Design phase. Contracts are described by intent and data shape; exact API
> endpoints get finalized when both sides are built.
>
> **Tags:** [LATER] deferred until dependency ready · [OPEN] undecided · [TBD] awaiting input.

---

## 1. Architectural principle

- RolliTime, RolliSuite (RS / the ERP), RolliWorking (RW), and the authenticator app are
  **separate systems with separate databases.**
- They integrate via **defined API contracts**, never by sharing a database.
- RolliTime **owns** its testing data and the dial-photo library; other systems read from
  it (and in defined cases push to it) through contracts.
- This separation matters especially now: the ERP suite is being **rebuilt in Emergent**,
  and this document is the contract both sides build against so nothing is lost in the
  rebuild.

---

## 2. Systems & roles

| System | Role | Relationship to RolliTime |
|--------|------|---------------------------|
| **RolliTime** | Watch accuracy / power-reserve testing | Owns testing data + dial-photo library |
| **RolliSuite (RS)** | ERP / client files | Source of truth for caliber & reference data; consumes test results + photos |
| **RolliWorking (RW)** | Workflow / process tracking; knows watchmaker assignment | Receives status; consumes testing data for watchmaker analytics |
| **Authenticator app** | Watch authenticity verification (genuine vs. counterfeit) | Mines the dial-photo library as a data source |

---

## 3. RolliTime ↔ RolliWorking (RW)

### 3.1 RolliTime → RW (push: status)
- When a watch is **scanned at the photo/testing station**, RolliTime signals RW to set
  that watch's process status to **"in testing."**
- **Trigger:** the photo-station scan is a deliberate, intentional action (good — not a
  stray background event). **[Note]** Because this writes a status change into a live
  production workflow system, the trigger should be deliberate and ideally confirmed, so a
  mis-scan doesn't wrongly flip a watch's status in RW.

### 3.2 RW → (pull: testing data)
- RW **pulls testing data from RolliTime** to power its **watchmaker analytics** (RW knows
  which watchmaker is assigned; RolliTime does not and does not need to).
- Data RW consumes: amplitude, power reserve, accuracy, pass/fail vs. tolerance — keyed by
  **caliber** and **watch (ref-serial)**.
- Example RW reports (built in RW, not RolliTime): "avg amplitude for caliber 2130 per
  watchmaker," "avg power reserve for caliber 3035 per watchmaker," staff quality patterns.
- **Division of labor:** RolliTime produces *testing* analytics (by caliber/watch); RW does
  *watchmaker* analytics by cross-referencing its assignment data with RolliTime's results.

---

## 4. RolliTime ↔ RolliSuite (RS / ERP)

### 4.1 RolliTime → RS (push/serve: results + photos)
- RolliTime **exposes test results** so they land on the **client's file in RS** (printable
  client report data — see `SPEC.md` §9).
- RolliTime **serves the dial-photo library** to RS (see §6).

### 4.2 RS → RolliTime (pull: caliber & reference data)
- **RS is the source of truth for caliber and reference-model data** (ref# → caliber,
  caliber specs, min tolerances) **for every Rolliworks app — RW and RolliTime both
  consume from RS.** Today RS and RW each maintain their own reference libraries and have
  drifted; the rebuild consolidates these into RS-authoritative shared reference data
  (see `REBUILD-PREREQUISITES.md` item #2).
- **Bootstrap:** RolliTime is **seeded initially with a CSV** of this data.
- **Going forward:** RolliTime **pulls this data from RS as the authoritative source**
  (live lookup / sync), replacing the CSV seed. *(This resolves the earlier open
  CSV-vs-live question: CSV to start, RS authoritative thereafter.)*

---

## 5. RolliTime ↔ Authenticator app

- The authenticator app verifies **watch authenticity** (genuine vs. counterfeit) and needs
  a **large corpus of dial photos** as a data source.
- It **mines RolliTime's dial-photo library** (read-only consumer — see §6).
- Throughout testing, RolliTime captures dial photos for many reasons (section captures,
  reconcile readings, open/close brackets, video frames); the auth app draws on this
  accumulated library.
- **[Quality note]** Authentication likely needs **higher image detail** than internal
  record-keeping. This interacts with the video-vs-stills tradeoff (`SPEC.md` §4.3): video
  frames are lower-resolution than dedicated stills. If the auth app needs high-detail dial
  images, consider capturing some **full-resolution stills** alongside the video. Flagged,
  not yet decided.

---

## 6. The dial-photo library (shared resource)

RolliTime's dial photos are a **shared asset** consumed by RS (client files), the
authenticator app (authenticity data), RolliTime's own UI, and — future — dial-reading
model training. Disciplined capture serves all of these at once.

### 6.1 Exposure
- Served via an **API** that returns photos + metadata for a given watch (by **ref-serial**).
- Read-only for external consumers (RS, authenticator app). RolliTime owns writes.

### 6.2 Metadata per photo
Each photo travels with: **ref-serial**, **caliber**, **capture timestamp**, **entered dial
time** (where applicable), **section/orientation**, **cycle**, and capture context (e.g.
section capture vs. open/close bracket vs. reconcile).

### 6.3 Retention
- Photos are **never discarded** (also serves future AI training — `SPEC.md` §5).
- Stored full-resolution where possible; library must stay **queryable** by ref-serial and
  caliber.

### 6.4 Access control
- **[OPEN]** Access/authorization model for the library (API keys, per-system permissions,
  who can read what). To be defined with the Emergent rebuild. See cross-cutting Q1/Q3 in
  `FUTURE-INTEGRATIONS.md`.

---

## 7. Data-shape requirements (so all consumers can query cleanly)

RolliTime must store and expose its data keyed for the above contracts:
- **By watch** (ref-serial) — the universal join key across all systems.
- **By caliber** — for RW analytics and RS reference linkage.
- **By metric** (amplitude, power reserve, accuracy, pass/fail) — discrete, queryable fields.
- **Photos** keyed by ref-serial + the metadata in §6.2.

Honoring this from the start (even before consumers are built) keeps every integration a
straightforward read rather than a later rework. Same principle throughout: build the core
so the data is cleanly queryable; the integrations then plug in.

---

## 8. Dependencies & open items

- **[LATER]** Emergent rebuild of the ERP suite (RS / RW) — exact API endpoints finalized
  then; this doc is the contract both sides build against.
- **[LATER]** Authenticator app — consumer of the photo library; build/timing TBD.
- **[OPEN]** Photo-library access/authorization model (§6.4).
- **[OPEN]** Whether to capture full-resolution stills (alongside video) for auth-app image
  quality (§5).
- **[RESOLVED]** Caliber/reference data: CSV seed now, **RS as source of truth** going
  forward (§4.2). Consolidation of today's RS+RW drift tracked in `REBUILD-PREREQUISITES.md`.
- **[NOTE]** RolliTime→RW status push writes to a live workflow system — deliberate/
  confirmed trigger (§3.1).
