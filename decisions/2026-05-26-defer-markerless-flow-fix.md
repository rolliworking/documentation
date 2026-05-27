---
silo: 10
date: 2026-05-26
decision_owner: user (Mike)
status: deferred — awaiting operational evidence
last_validated: 2026-05-27
---

# Defer Markerless Flow Misclassification Fix

## Context

Silo 10 research surfaced that ~10% of post-rollout estimates (May 2 onward) lack service markers and are silently classified as `BAND_ONLY_FLOW` regardless of actual work content. Of 15 markerless estimates sampled from the last 14 days:
- 6 were legitimate watchmaker jobs ($200-$2,000+) misclassified
- 3 were precious-metals work (PM-Solid Gold Bracelet)
- 6 were thin/draft/single-line estimates including a beacon test artifact

See `known-gaps/markerless-estimates.md` for full data.

Cursor traced the blast radius if a watch job is misclassified:
- Receive-watch UI flips to band-only mode
- Label wizard skips watch label
- RW creates wrong `job_components` (bracelet only, no watch_head/case)
- RC customer portal shows band timeline instead of watchmaker stages

## Decision

**Defer the fix.** Do not pick a direction (enforce markers / smarter detection / parts-department column) until operational evidence is gathered.

## Reasoning

The blast radius in code is large. The blast radius in actual operations is unknown. Questions that determine priority:

1. **Have operators complained about Item Type flipping to band-only on watchmaker jobs?** If yes → urgent. If no → maybe they're manually toggling Item Type and absorbing the friction silently.

2. **Have customers reported wrong timeline on the portal?** If yes → urgent. If no → maybe customers don't look at the portal closely, or workarounds exist.

3. **Have shop floor techs reported missing watch components on watch jobs?** If yes → urgent. If no → maybe techs route work informally and don't rely on component dots.

Without those answers, picking a fix direction risks:
- Building a system that solves the wrong problem
- Adding operator workflow friction (e.g. mandatory marker dropdown) that operators resent more than the bug they were silently working around
- Burning days of dev work on a problem that wasn't actually hurting anyone

The user has been burned 100+ times by false summits in silos 7-10. This decision is explicit pushback against pattern-matching to "obviously high-impact code bug → must fix immediately." First gather evidence the bug is hurting people in production.

## What "operational evidence" means

The user will ask three specific questions to staff:

1. *Show me the last band-only flagged job that you think was actually a watch job. How did you catch it?*
2. *Have any customers complained about wrong information on their portal timeline?*
3. *On the shop floor — do techs work from the component dots, or do they ignore them and route work some other way?*

Answers shape silo 11+ priority.

## What would escalate this

| Signal | Priority |
|--------|----------|
| Operator reports "the form keeps flipping my watch jobs to band-only" | Urgent (silo 11) |
| Customer complaint about wrong timeline | Urgent (silo 11) |
| Tech can't find watch_head component for a watch job | Urgent (silo 11) |
| No reports + no customer complaints + no shop floor issues | Stays deferred indefinitely; treat the misclassification as cosmetic |

## What would deprioritize this further

If markerless adoption climbs to 98%+ in the coming weeks, the 10% gap shrinks to 2% — possibly below the threshold of "worth structural change for." Re-evaluate at silo 12 or 13.

## Alternatives considered for fix direction (when fix is reactivated)

| Direction | Effort | Risk |
|-----------|--------|------|
| Enforce markers at estimate save (validation) | ~2 hrs code + UX | Operators select wrong marker to satisfy validation |
| Smarter flow detection from line content | ~4-6 hrs + ongoing | Pattern matching against drifting conventions |
| Add `department` column to `parts` table | ~1 hr code + variable backfill (~5,000 parts rows) | Backfill labor; durable once done |

User leaned toward Option C (parts.department) if forced to pick, but no decision committed.

## Coupled decisions

- `prefix-mismatch.md` cleanup is coupled — if option C is chosen, the dead `service_subcategories` matchers can be removed simultaneously
- `rw-data-blind-spots.md` rendering work may proceed independently; doesn't depend on flow accuracy
- RS→RW data contract redesign (silo 11 candidate) may incorporate flow as a first-class field, partially mitigating wrong-flow-on-RW even if RS classification stays imperfect
