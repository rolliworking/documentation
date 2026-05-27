---
silo: 6
last_validated: 2026-05-14
---

# Fixture — Estimate 24185 (multi-item bracelet swap bug)

## Identity

- Customer: Geoff Brenner (geoffbrenner@me.com)
- Estimate number: 24185
- Service codes: `[B]` × 2 (multi-item band-only)
- Line items:
  - Line 1: `BR-OY-PIN` — Bracelet Repair SS Bracelet — Steel parts
  - Line 2: `BR-OY-TT-PIN` — Bracelet Repair Two Tone — Two Tone parts

## The bug it exposed

After Stage 3 intake, `client_property` rows:
- item_index 1: bracelet_description "20mm Two Tone Oyster 78203" (TWO TONE) but ref `BR-OY-PIN` (STEEL line)
- item_index 2: bracelet_description "20mm Steel Oyster 93150" (STEEL) but linked to `BR-OY-TT-PIN` (TWO TONE line)

Descriptions swapped vs estimate sort_order. Operator typed each item's description manually without pre-fill.

## What it validates

- Plan A bracelet swap fix (ClientWatchEntryPage.tsx:618-638 pre-fill effect)
- Multi-item BAND_ONLY intake flow
- sort_order-based item resolution

## State after silo 6

Plan A fix applies forward-only. 24185's existing client_property rows are NOT retroactively corrected. Re-receiving would trigger correct pre-fill.

## RS payload contract investigation

RS does not persist outbound RW intake payloads (see `known-gaps/rs-no-outbound-audit-log.md`). Cannot verify what was actually sent for 24185 from RS side.
