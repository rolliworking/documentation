# Build — `rc-timeline-status` Endpoint

**→ Paste into: RS chat**

---

## Surgical fix only

Touch ONLY the files needed for this specific change. Don't refactor unrelated code. Don't add "while we're here" improvements. Minimal blast radius. If you spot other issues, flag them at the end but don't fix them.

## Goal

Build a new edge function `rc-timeline-status` that RC's client portal can call to display real-time status. RC will fetch on render, no persistent sync needed.

## Build per your brainstorm recommendation (#4 + #13)

### Phase 1: New edge function

**File:** `supabase/functions/rc-timeline-status/index.ts`

**Method:** POST  
**Auth:** Bearer `RC_API_KEY` (new secret — generate and add to env)  
**Body:** `{ estimate_number: string }`  
**`verify_jwt = false`** (API key gated, same pattern as `rw-job-status`)

**Logic:**
1. Validate API key
2. Validate `estimate_number` is provided
3. Run the timeline derivation logic from `JobHistoryDialog.tsx:450-700` to build the full `TimelineEvent[]`
4. Return a SMALL whitelisted DTO — NOT the raw event array

**Return shape:**
```json
{
  "ok": true,
  "estimate_number": "E26041",
  "current_step": "rw_in_progress",
  "current_step_label": "In Progress",
  "completed_steps": [
    "request",
    "estimate_created",
    "estimate_sent",
    "shipping_label_sent",
    "package_received",
    "intake_photos",
    "items_received",
    "labels_printed",
    "inspection_photos",
    "inspection_complete",
    "rw_in_queue"
  ],
  "next_step": "rw_in_testing",
  "next_step_label": "In Testing",
  "updated_at": "2026-05-17T19:30:00Z"
}
```

If estimate not found: return `{ ok: false, error: "estimate not found" }` with 404.

### Code organization

Per your brainstorm: extract the timeline derivation from `JobHistoryDialog.tsx:450-700` into a shared helper (e.g., `src/lib/timeline-derivation.ts` or `supabase/functions/_shared/timeline-derivation.ts`). Both the dialog AND the new endpoint should use the same function — single source of truth.

If extraction is too risky for tonight, you can duplicate the logic. But flag it as tech debt.

### Caching

Inside the function, use `cache_get` / `cache_set` with key `rc-timeline-status:{estimate_number}` and TTL 60 seconds. Avoids hammering on rapid portal page views.

### Phase 2: Cache invalidation hook (optional tonight)

In `job-status-update/index.ts`, after the existing `job_activity_log` insert, add a fire-and-forget call:

```ts
// Invalidate RC cache for this estimate (fire-and-forget, don't await)
fetch(`${SUPABASE_URL}/functions/v1/rc-timeline-status-invalidate`, {
  method: "POST",
  headers: { "Content-Type": "application/json", "Authorization": `Bearer ${RC_API_KEY}` },
  body: JSON.stringify({ estimate_number })
}).catch(() => {});
```

The invalidate endpoint just does `cache_delete('rc-timeline-status:' + estimate_number)`.

If Phase 2 is too much for tonight, ship Phase 1 alone. 60s cache staleness is acceptable.

## Out of scope

- No new persistent tables
- No cron jobs
- No outbox patterns
- No RC-side changes (separate work)
- No timeline UI changes in RS

## Acceptance criteria

1. New `rc-timeline-status` endpoint deployed
2. Bearer auth via `RC_API_KEY` enforced
3. Returns the whitelisted DTO shape above
4. Works for an existing estimate (test with E26041 or whichever Brian Poulson has)
5. Returns sensible 404 for unknown estimate_number
6. 60s cache to avoid load issues

## Test command

After deploy, share this for me to test:

```bash
curl -X POST https://djbjwcoddddywkgljuja.supabase.co/functions/v1/rc-timeline-status \
  -H "Authorization: Bearer <RC_API_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"estimate_number":"E26041"}'
```

(Or whichever estimate is current for testing.)

## Tell me when ready

After deploying, tell me:
- The endpoint URL
- The `RC_API_KEY` value (so I can pass to RC)
- Any tech debt flags or concerns

---

End of prompt.
