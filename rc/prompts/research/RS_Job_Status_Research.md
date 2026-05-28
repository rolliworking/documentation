# Research — RS Job Lookup Features and RW Integration

**→ Paste into: RS chat**

---

# Research mode — no code changes, investigate only

## Context

I want to understand what RS already knows about jobs, status, and the RW link. Before building automation to push status to RC, I want to know what's already wired up so we don't reinvent.

## Investigate

### 1. Job lookup features in RS

Walk through every place in RS where staff can "look up a job" or see job status:

- Where in the UI? List the pages/components
- What data does each lookup show?
- Does any lookup pull live data from RW? Or just cached/snapshot from RS DB?

### 2. RW → RS data flow

There's already a connection between RW and RS. Document it:

- What edge functions exist for RW communication? (e.g., `rw-job-status`, `job-status-update`)
- What direction does data flow? RW→RS, RS→RW, or both?
- When does RW push to RS? On every status change? Cron? On-demand?
- What table(s) on RS hold RW-derived data? (`job_activity_log` mentioned earlier — is there more?)

### 3. The `rw-job-status` function specifically

The earlier research mentioned this. Open the file and explain:
- What's the API it hits? (RW's URL?)
- How is RW status fetched? Pull on demand? Cached?
- What data structure does it return?
- Where in RS is it called?

### 4. Job status data shape

What's the actual shape of "job status" as RS understands it?

- Just current step? (string)
- Full timeline of completed events?
- ETA/target dates?
- Photos / attachments tied to status?
- Cost / parts info?

### 5. The bidirectional possibility

Is there any existing infrastructure that could be used in reverse — i.e., RS pushing TO somewhere instead of pulling FROM RW?

- Any webhooks RS already emits?
- Any "subscribe to estimate changes" mechanism?
- pg_net or other extensions enabled?

### 6. Performance and freshness today

When staff opens a job in RS today:
- Is the timeline computed on-the-fly (every render)?
- Cached? For how long?
- What's the typical render time?
- If `rw-job-status` is called per render, is RW being hammered?

## Report

Write a structured findings doc with:
- Inventory of existing lookup mechanisms
- Data flow diagram (RW ↔ RS)
- File references for key edge functions
- Performance characteristics
- Any "free" capability we could leverage

Don't apply changes. Research only.

---

End of prompt.
