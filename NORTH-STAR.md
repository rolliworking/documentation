# North Star — Vision & Goals for the Rolliworks App Suite

> **What this document is:** The anchor we re-read to stay oriented through the entire
> rebuild. It holds the *why* — the vision, the strategy, and the principles — so that
> module-by-module execution never drifts from the goal. Every rebuild decision should be
> checkable against this doc: "does this serve the North Star, or are we drifting?"
>
> **Status:** Living document. Add to it; don't let it go stale. When a decision changes the
> strategy, record it here, not just in a chat.
>
> **Audience:** Michael, and every tool/collaborator in the pipeline (Qwen, Opus, Cursor,
> Emergent). This is the orientation layer they should be pointed at first.
>
> **Scope note:** This is the *operational* vision doc. Confidential exit specifics
> (valuations, individuals, deal terms) live separately and are deliberately kept out of here.

---

## 1. The one-line goal

Build an integrated suite of applications where **fixing operational bottlenecks captures
valuable byproducts, and reusing that data improves the next workflow** — so the business
runs better today and accumulates hard-to-replicate assets for the long term.

If a piece of work doesn't either fix a real bottleneck or capture/route a byproduct, question
whether it belongs in scope right now.

---

## 2. The core thesis: byproducts as assets

The most valuable things this suite produces are **not** the features. They are the data and
assets that accumulate as a *side effect* of doing business — at essentially zero marginal
cost, and nearly impossible for a competitor to replicate.

The discipline this creates: when documenting or rebuilding any module, the high-value
question is not only **"what does this module do?"** but **"what does it produce as a
byproduct, and which other part of the suite should consume it?"**

Examples of the byproduct pattern across the suite:

- **Dial-photo library** — captured because testing/inspection *needs* photos, but the same
  library feeds the authenticator app, becomes model-training data, and lands on the client
  file. One operational act, multiple compounding assets.
- **Clean client conversation history** — a side effect of handling customer messages well;
  becomes the substrate for knowledge capture and faster future service.
- **Test/performance data** — produced by running watches through testing; feeds watchmaker
  quality analytics and dispute resolution.
- **Captured questions and decisions** — the residue of daily operations; becomes knowledge
  infrastructure over time.

**Design implication:** build the core so the data is cleanly queryable and the seams to other
consumers exist *from the start*, even when the consumer isn't built yet. Honoring this early
turns every future integration into a straightforward read instead of a painful rework.

---

## 3. The suite — what each app is and how they tie together

The apps are **not** independent projects. They are one ecosystem expressing the same
underlying assets. The integration *is* the value.

| App | Role | Key byproduct it produces / consumes |
|-----|------|--------------------------------------|
| **RolliConnect (RC)** | Customer-facing system (estimates, messaging, client portal) | Produces clean client conversation history + accurate client-facing status |
| **RolliSuite (RS)** | Internal ERP / client files | Source of truth for caliber & reference data; consumes test results + photos |
| **RolliWorking (RW)** | Watchmaker workflow / process tracking | Knows watchmaker assignment; consumes testing data for quality analytics |
| **RolliTime (RT)** | Watch accuracy / power-reserve testing | Owns testing data + the dial-photo library; pushes status to RW |
| **Authenticator app** *(future)* | Genuine-vs-counterfeit verification | Mines the dial-photo library as a data source |
| **M3KE** *(future)* | Knowledge infrastructure / chatbot | Consumes captured questions & knowledge across the suite |

**Architectural principles holding the suite together:**

- **RS + RW share one backend; RT stays a separate system with its own database.** RT
  integrates via defined API contracts, never a shared DB read. The separation is deliberate —
  a clean seam, not isolation.
- **RS is the single source of truth** for caliber and reference-model data, consumed by RS,
  RW, RT, and future modules.
- **Apps talk through contracts, not by reaching into each other's data.** The shared backend
  and the API contracts *are* the byproduct-routing layer.
- **Reserve seams for what isn't built yet** (authenticator, M3KE, and other future modules)
  so today's choices don't foreclose tomorrow's assets.

---

## 4. Operating model — how we actually build

The role here is **product owner directing an AI engineering team**, not pseudo-developer.
The scarcest, most valuable input in the whole rebuild is *product truth*: how the apps are
really used, what the bottlenecks are, and how the tech integrates with the physical shop
workflow. Everything else (architecture, code, tests) is a commodity the tools produce.

**The pipeline (each tool has one job):**

1. **Existing apps + docs + SQL** — source of truth for how things really work.
2. **Local models (Qwen primary, on the Corsair)** — the context refinery. Reads code,
   produces structured markdown: module summaries, workflows, dependencies, gap analyses.
   Cheap, unlimited, local.
3. **Opus** — the master planner. Turns clean markdown into rebuild briefs, architecture
   decisions, and acceptance criteria.
4. **Cursor / Emergent** — execution. Bounded, well-specified module rebuilds — never "rebuild
   the app" in one vague shot.
5. **Michael** — reviews output against shop reality; supplies the judgment the data can't.

**Non-negotiable execution disciplines:**

- **One bounded module at a time.** The failure mode is spreading effort thin and never
  finishing a slice. Smallest usable rebuild first.
- **Keep the live apps running.** Replace one workflow at a time, cut over when each slice is
  ready. Never break daily operations for a rebuild.
- **Markdown is the system of record.** Decisions, workflows, gaps, and contracts live in the
  docs repo — not only in chats or in someone's head.
- **Account for the future, build for the present.** Document where future items plug in
  (placeholders/seams); don't try to build all of them now.

---

## 5. Current priorities (update this section as work progresses)

> This is the part that moves. Keep it honest about what's actually next.

**First production target: RolliConnect (RC).** Two bounded bottlenecks, both real daily
friction, both byproduct sources:

1. **Client portal process flow** — the portal shows a single linear "watch work" flow, but
   watches enter states that don't fit it (e.g. *in testing*, *uncased*), which confuses
   clients. Needs to represent non-linear states cleanly and support multiple/returning-client
   flows. *Byproduct:* an accurate client-facing status is itself a trust/credibility asset.
   *Placeholders needed:* in-testing state, uncased state, multiple-flows / returning-client
   capability.
2. **Message thread fracturing** — one client ends up with multiple fragmented conversation
   threads instead of one unified conversation. Needs reliable routing + thread merging.
   *Byproduct:* clean, unified conversation history that feeds knowledge capture and faster
   future service.

Everything else across the suite is, for now, something we can live with. Resist the pull to
expand scope until these are done.

**The wishlist** (28 items, mostly RS/RW/RC) is a *backlog and an architectural constraint
set* — parked for building, but consulted for designing, so today's choices leave room for it.

---

## 6. How to use this document

- **Before starting any module:** re-read §1 and §2. Ask "what's the bottleneck, and what's
  the byproduct?"
- **When pointing a tool at the code:** point it here first for orientation, then at the
  specific module.
- **When a decision feels big:** check it against the principles in §3 and §4. If it drifts
  from the North Star, that's the signal to pause.
- **When something changes:** record it here. A vision doc that isn't maintained stops being a
  North Star and becomes archaeology.

---

## 7. Decision log (append-only — newest at top)

> Log meaningful strategic or architectural decisions here so they don't get re-litigated.
> Format: date — decision — reasoning.

- *2026-06-07* — RC chosen as the first production app to refine/rebuild, scoped to the client
  portal process flow and message-thread fracturing. Reasoning: both are the only bottlenecks
  currently causing real daily friction, and both produce byproducts worth compounding (clean
  client status, clean conversation data).
- *2026-06-07* — RT used as the pilot to prove the local Qwen → markdown pipeline before
  pointing it at production apps. Reasoning: small, architecturally decoupled, low-stakes;
  proved the machine works on real code.
