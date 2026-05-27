---
silo: 10
date: 2026-05-27
decision_owner: user (Mike)
status: adopted; first implementation = this bundle
last_validated: 2026-05-27
---

# Establish Documentation Repo to Preserve Silo Context

## Context

After silos 7-10 (~100+ cumulative hours on receive-watch + related issues), the user observed:

> "every new chat knows less than the previous. we create code that often gets abandoned. by the end of the chat there better knowledge then we reach the end and do a hand off and the cycle starts again"

The pattern: each new Claude/Cursor chat starts with a sticky-note summary, spends the first 1-2 hours rebuilding context, makes progress, then ends with another sticky note that loses ~70% of the texture. Knowledge that was hard-won in the back half of one chat is unavailable to the next.

What survives handoff:
- Code commits (the *what* but not *why*)
- A short sticky note (lossy; like packing a moving truck into one box)

What dies:
- Theories ruled out and why
- Operational reality vs. code reality mismatches
- Texture of which research outputs were credible
- Methodology learnings

## Decision

Create and maintain a `rolliworking/documentation` repo to persist architectural context across silos.

Structure:
```
documentation/
├── README.md
├── architecture/        # overview, data flow, key concepts
├── known-gaps/          # one file per gap, with evidence and status
├── decisions/           # one file per major choice, YYYY-MM-DD-topic
├── silos/               # one file per silo close, with 7 standard sections
├── fixtures/            # test fixtures (beacons, sentinel data)
└── handoff/
    └── current.md       # live "what to know" doc for next chat
```

Each new chat is instructed: "Before anything else, read https://github.com/rolliworking/documentation/blob/main/handoff/current.md and any docs it references."

## Reasoning

**Why a separate repo, not docs in code:** Architectural memory (decisions, gaps, methodology) is orthogonal to the codebase. It changes on its own timeline, sometimes covers multiple repos (RS / RW / RC), and would clutter or fragment if scattered into each app's repo. A dedicated documentation repo is also easier for assistants to fetch via web_fetch / git clone.

**Why structured frontmatter (`silo`, `date`, `status`, `last_validated`):** Allows future chats to identify staleness. A doc dated 6 months ago with no recent `last_validated` should be re-verified before being trusted.

**Why "verified vs assumed" distinction in every doc:** Silo 10 burned hours on assumptions treated as facts. Codifying the distinction in doc format helps prevent recurrence.

**Why standard 7-section silo template:** Forces every silo to capture: shipped / learned / deferred / wrong / decisions / artifacts / open threads. Sections that don't apply can be marked "n/a" but can't be silently skipped.

## Risks named honestly

1. **Docs go stale.** Most engineering docs do. If silo close doesn't include 30 min of documentation update, the repo decays. Single biggest risk.
2. **Diminishing returns past ~10-20 docs.** First docs are high-value. Doc #50 might be noise. Discipline needed.
3. **User becomes responsible for honesty audit.** Claude can draft updates; user has to confirm them before commit. Otherwise the docs become claims, not truth.
4. **Other devs (if any) must read it.** For a solo dev + AI workflow, this risk is small.
5. **Chats might skip the read-first instruction.** Pattern needs reinforcement in every new chat opener.

## Maintenance cost

~30 min at each silo close to:
1. Update `handoff/current.md` to reflect changes
2. Create `silos/silo-N.md` with the 7 standard sections
3. Update affected `known-gaps/*.md` files
4. Add `decisions/YYYY-MM-DD-*.md` for any choices made

Estimated payback: each future chat starts ~60 min smarter on context regain. Net positive at 1-2 silos/week cadence.

## First implementation

Silo 10 docs (this bundle) — drafted at silo close, zipped for user to commit to `rolliworking/documentation` via a Cursor prompt included in the bundle's README.

Future silos: prompt template exists for any previous-silo chat to retroactively produce its own bundle. Older silos may produce thinner docs if their texture has decayed in the chat history.

## Alternatives considered

- **In-codebase ARCHITECTURE.md** — rejected; doesn't survive across the three repos cleanly, doesn't capture process/methodology learnings.
- **External notion / confluence / google doc** — rejected; harder for assistants to access programmatically, requires auth, doesn't integrate with git workflow.
- **Personal notes in user's local files** — rejected; not accessible to assistants, doesn't survive machine changes.

## Success criteria

After 3 silos using the repo:
- Does silo N+1 cost less context-regain time than silos before the repo existed?
- Are docs being updated at silo close, or decaying?
- Are assistants actually reading `handoff/current.md` when instructed?

If yes to all → keep the practice. If decay is happening → revisit format or maintenance cadence. If assistants ignore the instruction → strengthen the opener template.
