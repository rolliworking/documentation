---
silo: 7
date: 2026-05-14
status: applied to test branch, smoke test pending
last_validated: 2026-05-14
---

# Performance Report — Option A (first-name only) over Option B (identity overhaul)

## Context

Mike asked for "customer name and ref-serial#" on the Performance Verification Report. Investigation revealed:

- The report already stores and renders `client_last_name` (single text column) and `reference_number` + `serial_number` (separate text columns rendered as `"<ref> / <serial>"`).
- No `client_first_name` exists in schema or form.
- Ref-serial format is `<ref> / <serial>` (space-slash-space, full serial). Different from RW canonical `<ref>-<serial-last-3>` format.

Four scoping options surfaced:

- **Option A:** Add `client_first_name` only. Smallest possible change.
- **Option B:** Replace `client_last_name` with full identity (first/last/company), pull from joined `customers` row, render with fallback.
- **Option C:** Render from joins instead of snapshots. Stop storing identity fields on report row at all. Biggest change, fixes snapshot drift permanently.
- **Option D:** Add email/send delivery action. Independent of A/B/C.

## Decision

**Option A, narrowed.** Mike confirmed:
- Business clients still get a contact person name. Last name stays.
- "options A but last is only stays in place" — keep existing `client_last_name` column and plumbing entirely unchanged. Add first_name as additive.
- Ref-serial format scope creep rejected: "why are we changing or asking about this. leave the report as-is."

## Reasoning

1. Mike has business clients but they get attached to a contact person, so personal-name fields are the right model. No need for `display_name` or `company_name`.
2. Existing `client_last_name` plumbing is working — don't touch what works.
3. Snapshot drift (Option C's concern) is real but not what was asked about. Park as known gap, not a fix.
4. Delivery mechanism (Option D) is high-leverage but bigger scope, separate decision.

## What changed

- Migration: `ALTER TABLE performance_verification_reports ADD COLUMN client_first_name text` (nullable).
- `usePerformanceReport.ts` — payload interface extended.
- `PerformanceReportEntryPage.tsx`:
  - `IdentityState` interface extended.
  - `emptyIdentity()` updated.
  - SO prefill query extended: `customers(last_name)` → `customers(first_name, last_name)`.
  - SO hydration `setIdentity` pulls `first_name`.
  - Existing-report hydration pulls `client_first_name`.
  - Save payload includes `client_first_name`.
  - New "Client First Name" form field placed immediately before "Client Last Name".
- `PerformanceReportPreviewPage.tsx`:
  - IdentityTable Client row: `[r.client_first_name, r.client_last_name].filter(Boolean).join(' ') || '—'` — graceful NULL fallback for historical rows.

## Verification status

- Lovable Apply confirmed by user.
- Migration existence NOT directly verified via SQL in-session (recommended check: `SELECT column_name FROM information_schema.columns WHERE table_name = 'performance_verification_reports' AND column_name = 'client_first_name';`).
- Smoke test on real SO with populated customer first_name NOT yet performed.
- Smoke test on historical report (NULL first_name) NOT yet performed.

## What would change the decision

- If Mike has substantial business-only clients with no contact person, `company_name` becomes worth adding (Option B).
- If snapshot drift produces a customer-facing incident (wrong name on a delivered report), Option C becomes more urgent.
- If manual report delivery becomes more painful (volume grows), Option D rises in priority.
