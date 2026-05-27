---
silo: 10
date: 2026-05-25
decision_owner: user (Mike)
status: adopted as standard pattern
last_validated: 2026-05-27
---

# Bulldozer Methodology — Pipeline-Trace Research Before Fix

## Context

Through silos 7-9, debugging followed a bug-of-the-day pattern: identify one symptom, theorize a cause, ship a fix, find the symptom unchanged or replaced by a new symptom. Each cycle took hours. Cumulative cost across the silos: 90+ hours with no resolution.

Underlying reason: each "fix" was treating a symptom that was downstream of the actual root cause. Earlier silos shipped commits a3e4a339, 70505bdf, cb322d0d during silo 10's evening — three real bugs fixed, but the form still rendered blank because none of them was the root.

## Decision

User invented and named the bulldozer methodology:

> "trace EVERY surface the beacon should appear on, identify what shows up and what doesn't, and produce a single summary table"

The pattern:
1. Pick a sentinel value (the beacon)
2. List every pipeline stage from data source to user-visible output
3. For each stage, document: code path, expected behavior, observed behavior, where the beacon appears or doesn't
4. Don't stop at the first break — keep tracing past every gap by mocking correct data if needed
5. Produce one summary table mapping all gaps in one pass
6. Then plan fixes in sequence

The opposing pattern this replaces: find first break, fix it, retest, hope for the best.

## Reasoning

**Why "don't stop at first break":** A pipeline with N gaps fixed one at a time costs N debug-fix cycles, each ~3-10 hours in this codebase. A pipeline traced in one pass with all N gaps identified costs ~1 research session plus N targeted fixes. The research session is cheaper than even one debug-fix cycle.

**Why "single summary table":** Forces the researcher (Cursor or Claude) to commit to a structured output. Prose investigation tends to bury findings; the table format forces every stage to have an explicit row.

**Why pipeline-trace specifically:** Most bugs in this codebase live in handoffs between layers (Supabase → React Query → useEffect → setFormData → render; or RS edge function → RW edge function → RW DB → RW UI). Tracing along the data path catches handoff bugs that don't appear when reasoning about any single component in isolation.

## How it was applied in silo 10

Two bulldozer-style research reports were produced, on RS-side and RW-side. Each report:
- Numbered every stage of the pipeline
- Listed expected vs. observed behavior at each stage
- Documented gaps and dead code
- Ended with "Known unknowns" / "Fields needed but not received" lists

Reports were then cross-validated by a second independent research pass (also using the bulldozer pattern but with different framing questions).

Result: the silo 10 root fix (`f2e1c0ba`, `.eq()` → `.or()`) emerged from understanding the URL-param vs. scan-path divergence — which only became visible by tracing both paths end-to-end and comparing.

## Conditions for using bulldozer

Use it when:
- A bug has resisted multiple targeted fixes
- The bug crosses a layer boundary (frontend ↔ DB, app ↔ app, etc.)
- Multiple writer or reader paths exist for the same data
- The symptom doesn't pinpoint a single component

Skip it when:
- The bug is contained to a single component with a clear stack trace
- A targeted fix takes 5 minutes to ship and revert if wrong
- The cost of full research exceeds the cost of guess-and-check (rare; the threshold is lower than instinct suggests)

## Lesson from silo 10

Bulldozer should have been applied at hour 10, not hour 95. Cost of the methodology itself: ~2-4 hours of Cursor / Lovable research time. Cost of not applying it: 85+ additional hours of debugging.

Pattern for future: at hour 5-10 of any cross-layer bug, force a bulldozer pass before any more fix attempts.

## Companion: ground-truth investigation

The bulldozer methodology depends on the researcher having access to real data. When Cursor / Claude couldn't run queries against the live DB, the bulldozer reports were partly speculative. In silo 10 a follow-up "ground-truth investigation" pattern emerged:

> Don't speculate about what's in the database. Run a query. Paste the raw result. If you can't run the query, say so and ask for it.

Documented separately if needed; for now, captured as part of the bulldozer pattern.

## Anti-pattern: false summits

User counted 50-100 "got it!" / "found it!" moments in silo 10 that turned out wrong. Each was a premature declaration that the bug was found, often based on:
- One log line that looked diagnostic
- One code path that looked suspicious
- One theory that explained one symptom but not others

The bulldozer's "produce a single summary table mapping every gap" is partly a defense against false summits. If the summary table is incomplete, the methodology isn't finished. If the methodology isn't finished, no claim of "found it" should be made.

Lesson for assistants (Claude, Cursor) helping in future silos: don't announce "found it" until the bulldozer table is complete and the proposed fix accounts for every row in the table.
