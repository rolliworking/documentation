## Silo 6 contributions (2026-05-12 to 2026-05-14)

### Verified shipped

- set_job_status SQL fix (qualified job_id)
- job_component_notes table (RW database, RLS gated on jobs.view)
- See `silos/silo-6.md` "Verified live" section

### Apply unverified (Lovable confirmed but plan-mode confusion surfaced late session)

- Shop floor geometry fix (VB_W 1240→1260)
- Bulk Assign watch-label scan fix (uses resolveJobFromScan)
- Backfill components "+" feature
- Job lookup popup scroll fix
- Plan A intake payload fixes
- Shop floor drag v1 (7 lock stations)
- Tech picker v2 (5 work-node stations + TechPickerDialog)
- dotStationFor override fix

**ACTION ITEM:** Verify each "apply unverified" item against production code in git. Either confirm shipped or move to backlog as pending-apply.

### Architectural findings (load-bearing)

- QBO push intentionally bypasses set_job_status (raw UPDATE via service role)
- Two parallel state machines (jobs.status, job_components.status)
- "Documented but unused" pattern recurring in codebase
- RS doesn't persist outbound RW intake payloads
- received_case was structurally hard-coded (fixed in Plan A)
- jobs.serial_number actually stores reference numbers (misnamed column)
- job_component_logs.assigned_to is uuid-typed but tech IDs aren't auth UUIDs

### New gaps documented

- `known-gaps/plan-vs-apply-confusion.md`
- `known-gaps/intake-payload-contract-drift.md`
- `known-gaps/rs-no-outbound-audit-log.md`
- `known-gaps/assigned-to-uuid-mismatch.md`

### Plans designed but not started

- Plan B: rw_intake_log table in RS (audit log for outbound payloads)
- Plan C: Structural intake changes (Stage-2 booleans, Confirm & Send UI, payload v2)

### Workflow conventions established

- "Surgical fix" framing = Mike's Approve trigger
- "Mode: Surgical investigation, NO code changes" for read-only investigation
- Plan-vs-apply distinction must be verified, not inferred
- Three-AI cross-check pattern (Claude + RW Lovable + RS Lovable + Mike as bus)

### Open questions

- Apply state of Plan A and shop floor drag/picker items
- EST-24187 approval mismatch (Mike says approved, DB says waiting_approval)
- Has band-only intake catch-up batch happened?
- Does RW caret-delimited parser tolerate unknown trailing fields? (Plan C blocker)
