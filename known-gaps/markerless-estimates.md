---
silo: 10
date_surfaced: 2026-05-27
status: deferred — awaiting operational evidence before scoping fix
last_validated: 2026-05-27
---

# Markerless Estimates → Silent BAND_ONLY_FLOW Misclassification

## What it is

`get_job_profile` SQL RPC computes flow primarily from `[X]` marker lines on `estimate_line_items.entered_part_number`. When no marker is present, the SQL falls through to a service-code prefix matcher (`W-%`, `B-%`, etc.) against `service_subcategories.service_code`.

In practice, `service_subcategory_id` is NULL on 100% of production line items. The prefix matcher returns nothing. The final `ELSE` branch in the rollup at `supabase/migrations/20260502221531_181dc491-cf2c-4c8f-8826-295087bc77cd.sql:218` is `'BAND_ONLY_FLOW'`.

Result: every markerless estimate resolves to `BAND_ONLY_FLOW`, regardless of what work it actually contains.

## Scope (production data, last 90 days)

| Metric | Value |
|--------|-------|
| Total estimates | 876 |
| Has marker | 240 (27%) |
| No marker | 636 (73%) |

The 90-day blended rate is dominated by pre-rollout data. Service markers were introduced 2026-05-02 (~25 days before this silo).

| Window | Total | Has marker | Rate |
|--------|-------|-----------|------|
| Current week (partial) | 18 | 17 | 94% |
| Week of 2026-05-18 | 81 | 75 | 93% |
| Week of 2026-05-11 | 92 | 80 | 87% |
| Week of 2026-05-04 | 64 | 58 | 91% |
| Rollout week (2026-04-27) | 73 | 13 | 18% |
| Pre-rollout | 0% |

**Post-rollout steady state: ~90-94%, stable. Active ~10% gap on newly created estimates.**

## What the markerless 10% looks like (15 estimates from last 14 days)

| Group | Count | Examples | Notes |
|-------|-------|----------|-------|
| Real watchmaker service jobs | 6 | EST 25436 ($1,763), 24921 ($742), 24909 ($1,945), 24889 ($2,237), 24887 ($835), 24903 ($198) | **Genuine misclassification.** Full watchmaker estimates classified as BAND_ONLY. |
| Gold / precious metals | 3 | EST 25173 ($2,717), 25163 ($2,907), 24917 ($2,328) | All use `Gold-Price` + `PM-Solid Gold Bracelet`. Marker vocabulary may not cover PM cleanly. |
| Thin / draft / single-line | 6 | EST 25533 (beacon test artifact), 25257, 25210, 25156, 24932, 24907 | Includes a beacon test estimate. Some legitimately have nothing to mark yet. |

`estimates.created_by` is NULL on every row — can't identify if specific operators are the source.

## Blast radius if a watch job is misclassified (Cursor traced)

**Operator-visible:**
- Receive-watch UI auto-flips to band-only mode (different inspection type, hides watch identity fields)
- Save validation relaxed (brand not required, wrong custody flags)
- Label wizard skips watch label entirely (only QR labels print)
- "Job profile loaded: BAND ONLY FLOW" banner shows

**Shop-visible (RW):**
- `job_components` rows created wrong: bracelet only, no `watch_head` or `case` row
- Shop Floor / Work Queue / StationScanner depend on component records
- Watchmaker bench tracking missing or requires manual backfill

**Customer-visible (RC):**
- Portal timeline shows band workflow steps (Polish & Clean, Awaiting Start) instead of watchmaker stages (In Queue, In Progress, In Testing)
- No parallel watch/band track display
- Driven by `get_job_profile.flow`, not `jobs.flow`

**No impact (dead reads):**
- `jobs.flow` stored on RW — no reader
- `pending_print_jobs.flow` — metadata only
- `normalizeFlow` fallback to `rwStatus.flow` — dead path

## Why fix decision is deferred

Code blast-radius is large but real-world symptom rate is unknown. As of silo 10 close:
- Have operators complained about Item Type flipping to band-only?
- Have customers reported wrong timeline?
- Have techs reported missing watch components on shop floor?

If yes to any → fix is urgent. If no → operators may be absorbing friction silently, OR the friction isn't materializing because workarounds exist (manual Item Type toggle, informal work routing on shop floor).

Decision deferred pending operator/customer feedback. See `decisions/2026-05-26-defer-markerless-flow-fix.md`.

## Three possible fix directions

| Direction | Effort | Risk |
|-----------|--------|------|
| Enforce markers at estimate save | ~2 hrs + backfill | Operators select wrong marker to satisfy validation |
| Compute flow from line content (parts metadata or `entered_part_number` patterns) | ~4-6 hrs + ongoing maintenance | Pattern matching against drifting conventions breaks repeatedly |
| Add `department` column to `parts` table; flow becomes `SELECT departments FROM parts JOIN line_items` | ~1 hr code + variable backfill (5,000+ parts rows) | Durable; backfill is the real cost |

## SQL queries for future reference

### Markerless rate by week
```sql
SELECT 
  DATE_TRUNC('week', created_at) as week,
  COUNT(*) as total,
  COUNT(*) FILTER (WHERE EXISTS (
    SELECT 1 FROM estimate_line_items eli
    JOIN parts p ON p.id = eli.part_id
    WHERE eli.estimate_id = e.id 
      AND p.item_type = 'service_marker'
  )) as has_marker
FROM estimates e
WHERE created_at > '2026-04-01'
GROUP BY week
ORDER BY week DESC;
```

### Markerless estimates from last 14 days (detail)
```sql
SELECT 
  e.estimate_number,
  e.created_at,
  e.status,
  e.total_amount,
  e.created_by,
  STRING_AGG(
    eli.entered_part_number || ' (' || COALESCE(p.item_type::text, 'no-part') || ')', 
    ' | ' 
    ORDER BY eli.sort_order
  ) as lines
FROM estimates e
JOIN estimate_line_items eli ON eli.estimate_id = e.id
LEFT JOIN parts p ON p.id = eli.part_id
WHERE e.created_at > NOW() - INTERVAL '14 days'
  AND NOT EXISTS (
    SELECT 1 FROM estimate_line_items eli2
    JOIN parts p2 ON p2.id = eli2.part_id
    WHERE eli2.estimate_id = e.id
      AND p2.item_type = 'service_marker'
  )
GROUP BY e.id, e.estimate_number, e.created_at, e.status, e.total_amount, e.created_by
ORDER BY e.created_at DESC;
```
