# Research — Email Templates Structure (Shared vs Per-User)

**→ Paste into: RC chat (AFTER Vienna is added as responder)**

---

# Research mode — no code changes, investigate only

## Context

I want responders (like Vienna) to be able to create their OWN email templates without making them admin. Need to understand the current structure before deciding whether to extend.

## Investigate

### 1. Current email template structure

- What table holds email templates? (`email_templates`? something else?)
- What columns does it have?
- Is there an `owner_id` / `created_by` / `user_id` column?
- RLS policies — who can SELECT, INSERT, UPDATE, DELETE?

### 2. How templates are used today

- Where in the CRM UI are templates created/edited? (page references)
- Where are they used (compose new message? Saved replies carousel?)
- When a responder sends a templated message, does it know which template was used?

### 3. Are templates SHARED or PER-USER today?

- If a user creates a template, can other team members see it?
- Or is each user's templates private to them?
- Does the existing UI suggest one model over the other?

### 4. Current admin-only vs everyone capabilities

- What can admins do that responders can't?
- Specifically around templates: can responders read templates? Just not create?
- Walk through the UI: which buttons/pages do responders see vs not see?

### 5. What it would take to enable per-user templates for responders

If templates are currently shared/admin-managed, what would change:
- Add `created_by` column (if missing)
- RLS: responder can INSERT their own, READ all (or just own?), UPDATE/DELETE own only
- UI: filter "My templates" vs "All templates"
- Or: keep shared model but allow responders to create

### 6. Recommendation

Based on findings, recommend ONE of:
- **Path A:** Keep shared model, just allow responders to CREATE (admins moderate)
- **Path B:** Per-user templates with shared visibility (everyone sees everyone's, but only owner edits)
- **Path C:** Strictly per-user (each user has their own private set)
- **Path D:** Hybrid (shared "official" templates + per-user "personal" templates)

What fits the current architecture cleanest? Time estimate for each?

## Report

- Current template schema + RLS
- How they're used in UI today
- Shared vs per-user assessment
- Recommended path with rationale
- Estimated effort

Don't apply changes. Research only.

---

End of prompt.
