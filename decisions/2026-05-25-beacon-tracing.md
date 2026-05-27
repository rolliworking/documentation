---
silo: 9
date: 2026-05-25
status: adopted — methodology now baseline for cross-app verification
last_validated: 2026-05-25
---

# Decision: Adopt Beacon Tracing for Cross-App Data Flow Verification

## Context

Across silos, "is the data flowing correctly?" has been guessed at via real customer data with ambiguous values. When Rolex Oval Jubilee pre-fills correctly, is it because the architecture works, or because "Rolex" happens to match the Brand pill literal?

## Decision

Create **beacon test fixtures** — known sentinel data with unique strings that can be traced through the pipeline.

## Implementation pattern

**1. Create curated parts row with unique values:**
- part_number: `BR-Tracking-Beacon` (or BR-Tracking-Beacon2 for separate beacons)
- brand: `Rolex` (matches production pattern to avoid synthetic display bugs)
- bracelet_model: `Tracking-Beacon-Model` (unique, traceable string)
- department: B, item_type: service

**2. Create estimate referencing the beacon part:**
- Line 1: `[B]` marker with unique description like `[B] Tracking-Beacon-Description-Est5`
- Line 2: beacon part_number with unique description

**3. Walk through full pipeline checking for beacon strings:**
- /inventory/parts (source)
- estimates + estimate_line_items
- /intake/receive-watch form pre-fill
- client_property after Save
- RS push payload (caret string)
- RW jobs, watches, job_components tables
- RW shop floor display
- RC customer portal timeline

**4. Build a break map:**

| Stage | Beacon present? | Where dropped if no |

Every drop point is a fix target.

## Why this works

- **Eliminates guessing.** "Tracking-Beacon-Model" appearing on shop floor = data flowed. Not appearing = bug at this stage.
- **No data collision with real records.** Sentinel strings are unique.
- **Permanent regression test.** The beacon estimate stays in test database; can re-verify after any rebuild work.
- **Composable.** Add more beacons for split flow, polish flow, etc.

## Rules of beacon design

- **Match production data shape for non-traced fields** (e.g. brand=Rolex). Otherwise synthetic values expose synthetic bugs that don't actually affect real customers (this happened with first beacon brand="Tracking-Beacon-Brand" exposing Bug A which doesn't matter for Rolex/Tudor jobs).
- **Use unique strings for traced fields.** "Tracking-Beacon-Model" is searchable across DB and code.
- **Keep multiple beacons separate.** BR-Tracking-Beacon, BR-Tracking-Beacon2, BR-Tracking-Beacon-Watch — each tests a different flow.

## When to use

- After any cross-app code change that affects data flow
- Before claiming "the architecture works"
- When debugging "where did this field go?" questions
- For the rebuild — verify new architecture passes the same beacon

## Current beacon state at chat close

- **Created:** `BR-Tracking-Beacon` and `BR-Tracking-Beacon2` parts rows curated
- **Estimate:** EST 25530 referencing BR-Tracking-Beacon, customer test hui, dropped off 2026-05-25 at 10:20 PM
- **Status:** Pre-fill rendering blank on receive-watch despite SQL confirming data correctness
- **Next:** Bulldozer prompt drafted with raw-shape inspection — send in silo 10

## Evidence

- Multiple beacon investigations during silo 9 (Lovable RS reports for EST 25524, EST 25526, EST 25530)
- SQL verification of beacon parts row + estimate join
- Conversation transcript showing methodology evolution from "test with real data" → "create sentinel" → "bulldozer the beacon"
