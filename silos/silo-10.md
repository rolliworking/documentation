---
silo: 10
date_range: 2026-05-25 evening to 2026-05-27 early morning
duration_hours: ~9 (session); ~95 cumulative on the underlying bug
status: shipped (root fix verified); RS→RW research complete; markerless flow decision deferred
last_validated: 2026-05-27
---

# Silo 10 — Receive-Watch Prefill Root Cause + RS→RW Data Contract Research

## 1. Topic

Closing the receive-watch blank-prefill bug that had been open across silos 7-9 (~90 hours), then mapping the RS→RW data contract for the next round of work.

## 2. What Shipped

All on `rollisuite` branch `test`. Merge to `main` pending regression on EST 25523 and 24818.

| Commit | Description |
|--------|-------------|
| `a3e4a339` | Bugs 1+2+3 prefill race fixes — `hasPrefilledRef` lock during prefill, effect-177 guard simplified to ref check, `setItemType('band_only')` inside prefill when `isBandOnly` |
| `70505bdf` | Bug 6 — destructive toast when Rolliworking sync skipped due to missing brand at label Done |
| `cb322d0d` | Bugs 4+5 — band-only RadioGroup guard against clobber; custom brand pills map to "Other" highlight |
| `f2e1c0ba` | **THE ROOT FIX.** URL-param `useQuery` at L255 of `ClientWatchEntryPage.tsx` migrated from `.eq('estimate_number', estimateParam)` to `.or()` with same normalization (`trim`, strip `EST-?\|E` prefix, strip leading zeros, `padStart(5)`) that `handleEstimateScan` uses. |
| _pending commit_ | Scan path hands off to URL-param flow via `setSearchParams` — reuses the curated prefill effect instead of duplicating logic. Verified working at chat close, ready to commit. |

## 3. What Was Learned

### The actual bug (one line of code)

`ClientWatchEntryPage.tsx:287` used strict equality:

```ts
.eq('estimate_number', estimateParam)
```

`handleEstimateScan` at L1036 used `.or()` with normalization. Two queries against the same DB, different lookup strategies, only one path worked. The strict-eq path returned null, `estimateData` stayed undefined, main prefill effect early-returned at `if (!estimateData) return`, form rendered blank. Page header showed "Linked to Estimate: 25530" because `formData.estimateNumber` gets populated by THREE different code paths, none of which prove `estimateData` loaded.

The 90+ hour bug was a single Supabase query operator.

### Architectural debt confirmed

- **9 `setFormData` paths** in `ClientWatchEntryPage.tsx` write to `brand` / `model` / `partNumber` / `bracelet_description`. Documented in `known-gaps/nine-setformdata-paths.md`.
- **3 estimate-lookup queries** with different select shapes — L255 useQuery, `handleEstimateScan` L1036, `useCustomers` L51.
- **Service code prefix matching is mostly dead code.** `service_subcategories` is 9 rows; production line items have `service_subcategory_id = NULL` 100% of the time. SQL `ILIKE 'W-%'/'B-%'/'P-%'/'PM-%'` matchers in `get_job_profile` never execute against real data. Documented in `known-gaps/prefix-mismatch.md`.
- **Markerless estimates silently resolve to `BAND_ONLY_FLOW`.** ~10% of post-rollout estimates lack markers; most are real watchmaker jobs being misclassified. Documented in `known-gaps/markerless-estimates.md`.
- **`entered_part_number` has ~7 overlapping conventions** in production. No canonical convention exists.
- **RW stores fields it never renders** — `jobs.bracelet_description`, `received_components`, `received_bracelet`, `received_case` are never displayed on ShopFloor / WorkQueue / BandKanban. Documented in `known-gaps/rw-data-blind-spots.md`.
- **Caret payload positions 8, 10, 11 are dead** — RW receiver explicitly skips them.
- **`jobs.flow` on RW is write-only** — never read by any RW UI today.

### The "order ticket to the kitchen" framing (user-introduced)

Receive-watch is the order ticket. Upstream (estimate, parts curation, markers, package-receive) exists to make the ticket complete. Downstream (RW, RC) reads the saved ticket. The form itself shouldn't be smart — it should display consolidated upstream truth and let the operator confirm what physically arrived.

This reframes future work: instead of patching receive-watch logic, fix the upstream sources that should populate it.

## 4. What Was Deferred

- Bugs 7-12 from the bulldozer report (RW receiver caret[8], ShopFloor + WorkQueue not rendering bracelet_description, RC portal not surfacing metadata, RC timeline `normalizeFlow` fallback, caret pos 10-11 ignored, stale `useSendToRolliworking` hook). Non-blocking now that receive-watch works.
- Markerless-estimate fix direction — see `decisions/2026-05-26-defer-markerless-flow-fix.md`.
- Refactor of 9 `setFormData` paths into a single curated prefill function.
- Migration of caret payload to named JSON.
- Reconciliation of expected vs. received components (no `awaiting_arrival` status anywhere in RW).
- Adding `bracelet_model` rendering to RW shop floor / work queue / BandKanban.
- Investigation of the specific 15 markerless estimates from the last 14 days to see if there's a pattern (operator, entry path, work type).

## 5. What Was Wrong (Dead Ends)

User counted 50-100 "got it!" / "found it!" false summits across the silo arc. Theories pursued and disproven:

| Theory | Why it was wrong |
|--------|------------------|
| Supabase embed shape mismatch (array vs object) | Embed was returning correct object shape |
| Matcher filter rejecting valid parts | Matcher never ran because effect early-returned |
| Race condition between effect-177 and prefill | Real but secondary; root was upstream of effect |
| Draft persistence overwriting prefill | No draft layer exists; ruled out entirely |
| BR-Tracking-Beacon being uncurated | Was curated correctly |
| Page header "Linked to Estimate" proves query worked | Wrong — that data comes from any of 3 paths |
| Trace logs missing from bundle | Were on disk; filter setting was hiding them |
| Real codes are `WM-`, `B-`, `P-`, `PM-` | Production codes are `W-SERV-...`, `BR-JUB-...`, `GAS`, `MEMO.`, etc. |
| Service_subcategories matcher is broken in production | Confirmed never executes — `service_subcategory_id` is NULL on every line |

**Shipped 3 commits (a3e4a339, 70505bdf, cb322d0d) before identifying the root cause.** They fixed real but secondary bugs. The actual `.eq()` vs `.or()` fix came after, as commit `f2e1c0ba`.

**Playwright + Cursor hit `/auth` redirect twice** during the arc, blocking diagnostic capability. Required manual screenshots for ground-truth verification.

## 6. Key Decisions

| Decision | Reasoning | Conditions that would change it |
|----------|-----------|-------------------------------|
| Adopt beacon strategy permanently | Sentinel-value parts row (`BR-Tracking-Beacon`) eliminates real-data contamination during testing. Worked tonight when nothing else did. | If parts table cleanup ever happens, beacon must be preserved. |
| Adopt bulldozer methodology | Research entire pipeline in one pass instead of bug-of-the-day. Validated by independent second researcher. | Should have been done at hour 10, not hour 95. Use it earlier next time. |
| Two independent research reports before any fix | RS and RW researched separately, then cross-validated. Surfaced different findings, both correct. | Mandatory pattern for any cross-app debugging from now on. |
| Defer markerless flow fix | Code blast-radius known to be large, but real-world symptom rate unknown. Need operator/customer feedback before scoping fix. | If operators report visible breakage on markerless estimates, escalates immediately. |
| URL handoff for scan path | Cheapest fix — scan path sets URL param to trigger existing useQuery, no duplicate matcher logic. | If URL param approach breaks for any entry path, fall back to extracting prefill into a shared helper. |
| Establish documentation repo | Preserve context across silo handoffs; stop losing texture every chat. | Maintenance cost — if docs aren't updated at silo close they go stale. |

## 7. Artifacts Worth Preserving

### Test fixtures
- `BR-Tracking-Beacon` part (parts table) — Brand=Rolex, bracelet_model=Tracking-Beacon-Model, department=B, item_type=service. **Permanent regression anchor.** See `fixtures/br-tracking-beacon.md`.
- EST 25530, 25531, 25532, 25533 — beacon test estimates with `[B]` marker on line 1. Marker-path tested; service-code-path NOT yet tested.

### Diagnostic logs (live in code at chat close)
All in `src/pages/intake/ClientWatchEntryPage.tsx`:
- `[PREFILL EFFECT ENTRY]` (~L789)
- `[BEACON TRACE]` (~L855)
- `[MATCHER CHECK]` (~L867)
- `BR LOOP` (~L868)
- `BRACELET PART DETAIL` (~L888)
- `POST-PREFILL formData` (~L988)
- `LINE ITEMS RAW`
- `ESTIMATE QUERY RESULT`

Future silos: don't strip these blindly. They're cheap and they unlocked silo 10's fix.

### SQL queries that produced ground truth
- Markerless estimate counting with rollout-date bucketing — see `known-gaps/markerless-estimates.md`
- Bracketed-value hidden-char check — `SELECT '|' || estimate_number || '|' FROM estimates ...`
- Service_subcategories full dump
- Top entered_part_number values across 5,362 line items

### Methodology patterns
- **Beacon strategy** — see `decisions/2026-05-25-beacon-strategy.md`
- **Bulldozer research prompt** — see `decisions/2026-05-25-bulldozer-methodology.md`
- **Ground-truth investigation prompt** — preferred over inference-as-fact

## Open thread for silo 11+

Next session entry point:
1. Regression-test the scan-path fix on EST 25523 and 24818
2. Merge `test` to `main` if regression clean
3. Decide direction on markerless flow misclassification (or defer further pending operational evidence)
4. Begin RS→RW data contract redesign — research is done, design phase pending decisions on which gaps to close first
