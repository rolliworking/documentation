---
silo: 6
discovered: 2026-05-14
status: documented, partially fixed (Plan A), structural fix deferred (Plan C)
last_validated: 2026-05-14
---

# Gap — RS→RW intake payload contract drifted from documentation

## The problem

Data Trickle infographic (May 10, 2026) documents required payload fields:
`estimate_number, watch_brand, watch_model, serial_number, service_codes, received_bracelet, received_case, flow, bracelet_description, received_components, dept flags`

Actual code in `src/hooks/useRolliworkingSync.ts` and `src/components/labels/IntakeLabelWizard.tsx`:

**Missing from payload:**
- `serial_number`
- `flow` (BAND_ONLY / SPLIT / STANDARD)
- `orphan_parked` (not in contract but implied)
- `label_ids` (not in contract but implied)

**Extra in payload (not in contract):**
- `email`, `phone`, `part_number`, `date`, `bracelet_model`

**Structurally wrong (fixed by Plan A):**
- `received_case` was hard-coded `true` (IntakeLabelWizard.tsx:523) — every intake lied
- `received_bracelet` derivation was wrong for band-only flows in ClientWatchEntryPage and ReceivePackagesPage

## Why it matters

- RC integration next week will plan against documented contract; needs to know reality
- Cannot debug historical payloads (RS doesn't log them — see rs-no-outbound-audit-log.md)
- `received_case = true` for band-only jobs means RW thought every band-only intake came with a case

## Mitigation

- Plan A (silo 6) fixed the two structural lies forward-going
- Documentation drift remains

## Permanent fix (Plan C — deferred)

Plan C scope:
- Add missing fields (serial_number, flow, orphan_parked, label_ids)
- Version-bump payload to `v2^...` prefix
- Stage-2 boolean persistence on package_arrival_scans
- Confirm & Send UI step before label print
- Coordinated RW edge function update to parse v2

Requires verifying RW's caret-delimited parser tolerates unknown trailing fields (unverified).

## References

- `IntakeLabelWizard.tsx:523` — hard-coded received_case (fixed in Plan A)
- `ClientWatchEntryPage.tsx:2545` — receivedBracelet derivation (fixed in Plan A)
- `ReceivePackagesPage.tsx:4100-4102` — receivedBracelet derivation (fixed in Plan A)
- `useRolliworkingSync.ts:107-127` — caret payload structure
