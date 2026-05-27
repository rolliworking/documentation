---
silo: 9
date: 2026-05-25
status: implemented mid-session
last_validated: 2026-05-25
---

# Decision: Migrate Architectural Work from Lovable Preview to Cursor + Localhost

## Context

Spent ~4 hours of silo 9 debugging Lovable's preview infrastructure. Diagnostic console.logs were on disk (Lovable confirmed) but never fired in browser despite hard refresh, cache disable, multiple computers, dev server restart, .vite cache wipe. Lovable's investigation concluded "stale layer between local Vite and Lovable preview proxy."

## Options considered

**Option A: Stay on Lovable preview, file support ticket**
- Pros: No setup change
- Cons: Blocked until Lovable platform team responds; recurring problem class

**Option B: Switch to Cursor IDE + local clones**
- Pros: Localhost dev server has no proxy/CDN cache; diff visible before commit; cross-app investigation in one query; Vite HMR is reliable
- Cons: 30-45 min setup; user takes on git workflow responsibility; new tool learning curve

**Option C: Use different AI tool (Gemini, GPT-5, Kimi)**
- Pros: Maybe different debugging style helps
- Cons: Doesn't address infrastructure issue; problem isn't model quality

## Decision

**Option B — Cursor + localhost.** Mike already had Pro account.

## Implementation

- Three repos cloned: `~/rolliworks/{rollisuite, rolliworking, rollicrm}`, test branch each
- `npm install` in each
- `.env.local` with `VITE_SUPABASE_URL` + `VITE_SUPABASE_PUBLISHABLE_KEY` (RS uses PUBLISHABLE, not ANON)
- Vite dev server at `http://localhost:8080`
- Git identity: `mike@rolliworks.com` / Mike Hui

## Outcome

Race condition fix diagnosed in ~15 minutes once on Cursor (vs 4 hours through Lovable preview cache). Fix verified locally before commit. Subsequent receiver Fix A+B and RC band-flow Phase 1 also shipped via Cursor in single session.

## Conditions that would change the decision

- If Lovable preview proxy infrastructure stabilizes (the underlying cache issue is fixed at Lovable's platform)
- If Cursor cost becomes prohibitive (currently included in Pro plan)
- If localhost workflow becomes too slow for fast iteration on new UI scaffolding (likely never — Vite HMR is fastest)

## Hybrid approach kept

Lovable retained for:
- New feature UI scaffolding (Lovable's plan-then-apply discipline still valuable)
- Quick prompt iterations where local testing overhead isn't worth it
- Spinning up new pages and supabase migrations

Cursor used for:
- Architectural work across multiple files/apps
- Debugging that requires reading multiple codebases
- Refactoring
- Anything where verification feedback loop matters

## Evidence

- 4-hour debugging session on Lovable preview cache (silo 9 transcript)
- Race condition diagnosed in 15 min after Cursor switch
- All three silo 9 shipped fixes built and verified via Cursor + localhost
