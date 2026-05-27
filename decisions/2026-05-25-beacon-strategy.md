---
silo: 10
date: 2026-05-25
decision_owner: user (Mike)
status: adopted permanently
last_validated: 2026-05-27
---

# Beacon Strategy — Sentinel-Value Test Fixtures

## Context

By silo 9, debugging receive-watch was contaminated by real data variability. Every test estimate had different brands, different bracelet models, different service codes, different operator behavior. When the form rendered wrong, it was impossible to tell whether:
- The bug was in the form
- The estimate was structured wrong
- The parts curation was incomplete
- The matcher was rejecting a legitimate part for a reason specific to that part's data
- All of the above

False summits were frequent because every "fix" worked on one estimate and broke on the next.

## Decision

User created `BR-Tracking-Beacon`, a permanent sentinel-value part row in the `parts` table:
- `part_number`: `BR-Tracking-Beacon`
- `brand`: `Rolex`
- `bracelet_model`: `Tracking-Beacon-Model`
- `department`: `B`
- `item_type`: `service`

All fields populated with values that don't exist anywhere else in the system. If the form displays "Tracking-Beacon-Model" in the Model field, the data flowed cleanly from `parts.bracelet_model`. If the form is blank, the flow broke.

User also created beacon test estimates (EST 25530, 25531, 25532, 25533) using BR-Tracking-Beacon on line 2 with `[B]` marker on line 1.

## Reasoning

**Why sentinel values:** No collision with production data. Searching for "Tracking-Beacon-Model" returns only beacon-related rows. Tracing pipeline output is unambiguous.

**Why permanent:** The fixture is also a regression-test anchor. Future changes to receive-watch / RS→RW push / RW shop floor / RC timeline can be verified against the same known-good test case. Without it, every test cycle requires constructing a fresh estimate and inferring intent from variable data.

**Why parts-table fixture rather than test-suite fixture:** Production code reads from production tables. A test-suite fixture would only catch bugs reachable from the test runner. The beacon lives in the same data layer the real form reads from.

## Alternatives considered

- Use a real customer estimate for testing — rejected (data contamination, customer-data sensitivity)
- Create a synthetic estimate without curating a parts row — rejected (didn't isolate the parts-table → form-field path)
- Use unit tests instead of live fixtures — rejected (90 hours of debugging weren't a unit-test problem; the bugs lived in the integration between Supabase + React + auth + caching)

## What this would change

The beacon would be retired if:
- The parts table is restructured such that curated fields move elsewhere — beacon would need to move with them
- Production data ever uses "Tracking-Beacon" naming — collision risk (unlikely)
- A more comprehensive test-fixture system is built that supersedes ad-hoc beacons

## Beacon limitations surfaced in silo 10

Silo 10 research (Cursor's second pass) found:

> "EST 25530 flow works in RS UI because `[B]` marker → `fallback_flow = BAND_ONLY_FLOW`, not because `B-%`/`WM-%` matching succeeded. That's why the beacon is a good fixture but doesn't exercise the prefix-mismatch path."

The current beacon exercises the marker path. A second beacon estimate (e.g. service codes `WM-1` + `B-2` with no marker) would exercise the prefix-matching path. This was discussed in silo 10 but determined to be premature — depends on whether prefix-mismatch is a real production problem or dead code (see `known-gaps/prefix-mismatch.md`).

## Pattern for future silos

When debugging integration bugs:
1. Build a sentinel fixture upstream of the bug
2. Trace the sentinel value through each pipeline stage
3. Note where it appears unchanged, transformed, or absent
4. The first stage where the sentinel is absent or wrong is the bug location

This pattern produced the silo 10 fix (eventually). It would have produced it faster if applied earlier.
