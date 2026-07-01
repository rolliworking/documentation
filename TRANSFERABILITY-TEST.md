# Transferability Test — Design Gate for Every SPEC

> **Purpose:** Ensure every rebuild decision reduces (never increases) dependence on Michael Hui or Michael Michaels as individuals. This is the design gate applied to every SPEC before it locks.
>
> **Owner:** Michael Hui (author of the exit thesis this test enforces)
> **Status:** Active doctrine — apply to every SPEC review
> **Created:** 2026-06-30

---

## Why this test exists

The exit thesis for Rolligroup rests on a specific proposition:

> Rolligroup must continue to function at 90% quality after Michael Hui and Michael Michaels transition out of daily operations. If it can't, the exit is limited to modest cash-flow multiples plus retention agreements. If it can, the exit is a strategic acquisition at platform multiples.

The 11-app ecosystem, the canonical data model, the AI/data modules, and every architectural decision documented in DECISIONS-REGISTRY.md are means to this end.

The **Transferability Test** is the enforcement mechanism. Every SPEC that fails the test is redesigned before it ships. Every SPEC that passes the test moves Rolligroup closer to acquirable state.

---

## The five test questions

Apply these questions to every SPEC before it locks. If the answer to any question surfaces a gap, the SPEC gets revised before build begins.

### Question 1 — The Absence Test

**"If Michael Hui was gone for 3 months starting tomorrow, could Rolligroup operations continue at 90% quality using this SPEC's outputs alone?"**

What this catches:
- Undocumented "Michael knows" business rules embedded in code
- Approval workflows that depend on Michael's specific judgment
- Escalation paths that only route to Michael
- Configuration that only Michael knows how to change
- Trade-offs that Michael made silently in code without documenting the reasoning

Pass criteria:
- All business rules are documented in the SPEC or referenced from HUMAN-ANSWERS.md
- Approval workflows have documented decision criteria, not "ask Michael"
- Escalation paths route to defined roles (owner, manager, watchmaker lead), not named individuals
- Configuration surfaces (feature flags, thresholds, promotion rules) are documented with rationale

### Question 2 — The Mike Test

**"If Michael Michaels was gone for 3 months, could authentication quality, watchmaker training, and technical judgment be maintained at current levels using this SPEC's outputs?"**

What this catches:
- Reliance on Mike's Richemont-trained eye for authentication calls
- Technical decisions that only Mike can validate (parts sourcing, movement compatibility, service recommendations)
- Watchmaker mentorship and training gaps
- Tribal knowledge about specific brands, references, or service techniques

Pass criteria:
- Any decision requiring Mike-level expertise is either:
  - Captured in RolliCurator's validated dataset with Mike as the named source
  - Documented in a decision tree or knowledge base staff can reference
  - Escalated to defined technical review criteria (not "text Mike")
- Training materials exist for onboarding new watchmakers to Rolligroup standards
- Authentication decisions (even manual ones) reference documented visual criteria, not "vibes"

### Question 3 — The Onboarding Test

**"If a new operations manager was hired tomorrow with no prior Rolligroup context, could they run daily operations from documentation alone within 30 days?"**

What this catches:
- Institutional knowledge that only lives in current staff heads
- Workflow decisions that require historical context
- Vendor relationships and preferences
- Customer relationship history and preferences
- The "why we do it this way" reasoning that never got written down

Pass criteria:
- Every operational workflow has a runbook
- Every business rule has documented rationale
- Vendor relationships are captured in a vendor management system with terms, contacts, preferences
- Customer notes and preferences are structured data, not free-form chat history
- The Q&A that would happen during onboarding is pre-answered in accessible documentation

### Question 4 — The Handoff Test

**"When Rolligroup ownership transitions to an acquirer, could the acquirer's team take over operations without a 6-month retention commitment from Michael Hui or Michael Michaels?"**

What this catches:
- Systems that "only work because Michael knows the workarounds"
- Data quality issues that require Michael's context to interpret
- Integrations that were built without documentation
- Business logic that lives in Michael's mental model of edge cases
- The dozens of small decisions that keep the business running smoothly

Pass criteria:
- All systems have architectural documentation sufficient for a new team to understand and maintain
- Data quality is auditable — any staff member can spot broken data
- Integration contracts (like ROLLICLOCK-TO-RS-CONTRACT.md) are documented for every cross-app connection
- Edge cases are captured as tests, error handlers, or documented policies
- Operations have SLAs and quality metrics that a new team could measure and maintain

### Question 5 — The Multiplication Test

**"If Rolligroup added a second location (e.g., RolliShop expansion), could this SPEC's outputs replicate cleanly to the new location without Michael Hui personally standing up each system?"**

What this catches:
- Location-specific hardcoded values
- Manual setup steps that require deep system knowledge
- Configuration that only Michael can generate
- Data that has to be manually seeded per location
- Deployment processes that aren't automated

Pass criteria:
- New location deployment is automated where possible
- Multi-tenant support is designed in from the start (or clearly deferred with a documented migration path)
- Configuration is data-driven, not code-embedded
- Setup runbooks exist for each app
- The RolliShop scaling model (multiple locations) is technically feasible with the current architecture

---

## What passes and what fails — examples

### Pass example

**SPEC-001 (Two-Stage Intake) with Transferability Test applied:**

Original design: "Receive Watch page verifies components; if variance detected, staff acknowledge and proceed."

Test 1 (Absence): PASS — variance handling policy documented in the SPEC, no Michael involvement needed.
Test 2 (Mike): N/A — this SPEC doesn't touch authentication.
Test 3 (Onboarding): PASS — the three-page structure (Shipping Receive + Drop-off + Receive Watch) is documented with staff-facing training material.
Test 4 (Handoff): PASS — the audit trail captures every variance decision with rationale. An acquirer's team can reconstruct decision history.
Test 5 (Multiplication): PASS — the two-stage pattern is location-agnostic. Second location deployment reuses the SPEC.

### Fail example

**Hypothetical SPEC for approval routing: "Estimates over $2,000 require Michael approval before customer notification."**

Test 1 (Absence): FAIL — if Michael is gone for 3 months, all estimates over $2,000 stall.
Test 2 (Mike): PASS
Test 3 (Onboarding): FAIL — new manager can't approve because approval is hardcoded to Michael.
Test 4 (Handoff): FAIL — acquirer can't complete the transition without Michael-level approval authority.
Test 5 (Multiplication): FAIL — second location can't deploy this without Michael personally in the approval loop.

Revised design: "Estimates over $2,000 require owner/manager approval. Approval authority is a role in the auth model, not a specific person. Approval criteria are documented (revenue tier, customer tier, risk factors)."

Now all five tests pass.

---

## How to apply the test

### During SPEC drafting

At the end of each SPEC draft, add a section:

```markdown
## Transferability Test Results

| Test | Result | Notes |
|---|---|---|
| Absence Test (Michael Hui gone 3 months) | Pass/Fail | [specific findings] |
| Mike Test (Mike Michaels gone 3 months) | Pass/Fail | [specific findings] |
| Onboarding Test (new manager, 30 days) | Pass/Fail | [specific findings] |
| Handoff Test (acquirer takeover) | Pass/Fail | [specific findings] |
| Multiplication Test (second location) | Pass/Fail | [specific findings] |

**Failures identified:** [list any Fail results and how the SPEC will address them]
**Fail-to-Pass revisions:** [describe what changed in the SPEC to address failures]
```

A SPEC does not lock until all five results are Pass or a documented waiver is granted.

### During BUILD

When a build slice ships, the Transferability Test is re-verified against the actual implementation. Does the built system pass the same tests the SPEC promised?

If implementation drift caused a regression, it's fixed before the slice is considered done.

### During ongoing operations

Every 3-6 months, a random sample of existing systems gets re-tested. Systems can degrade over time — new features can introduce hidden dependencies on specific humans. Regular testing catches this.

---

## What this test does NOT enforce

Being clear about scope:

**This test doesn't enforce:**
- Technical quality (that's covered by SPEC review, code review, and QA)
- Feature completeness (that's covered by acceptance criteria)
- Performance or scalability (that's covered by dedicated performance testing)
- Customer-facing polish (that's covered by UX review)

**This test only enforces:**
- Reduced dependence on specific individuals (Michael Hui, Michael Michaels)
- Institutional knowledge captured in the system, not in heads
- Operational continuity across ownership transitions
- Scalability to multiple locations

Other quality gates continue to apply. This is one more, focused on the exit thesis specifically.

---

## Waivers

Some SPECs will legitimately fail the Transferability Test in ways that can't be fully addressed in that specific SPEC's scope. For example:

- Authenticator V1 relies on Mike Michaels' training data. That's inherent — Mike IS the authoritative source until RolliCurator has captured enough for the model to be independent.
- Chain of custody video review may require Michael Hui's judgment on edge cases in year 1.
- Certain vendor relationships may require Michael Hui specifically until they're renegotiated with new leadership.

For these cases, a waiver is documented:

```markdown
## Transferability Waiver

**Test failed:** [which test]
**Reason waiver is acceptable:** [specific justification tied to timing or dependency]
**Waiver expiration:** [when the waiver must be revisited — max 12 months from waiver date]
**Path to future compliance:** [what has to happen for the test to pass in future — e.g., "RolliCurator dataset expands to include this decision domain"]
```

Waivers are tracked in `TRANSFERABILITY-WAIVERS.md` and reviewed quarterly. Expired waivers must be revisited or renewed. Waivers accumulating without resolution is a sign the exit thesis is being compromised.

---

## Application to specific upcoming SPECs

### SPEC-001 — Two-Stage Intake

Expected result: PASS all five tests. The two-stage pattern is inherently designed for staff-independent operation.

Watch items:
- Variance handling requires clear policy (not "ask Michael when it happens")
- Component type mapping requires documented reference tables (not tribal knowledge)

### SPEC-002 — Canonical Data Model

Expected result: PASS all five tests. Canonical structure is the opposite of "Michael's mental model."

Watch items:
- Reference data seeding must not require Michael's memory — RolliCurator planned to fill this over time
- Data promotion lifecycle (D-011) must have clear criteria, not judgment calls

### SPEC-005 — Cross-app auth model

Expected result: PASS all five tests. Auth model must be role-based, not person-based.

Watch items:
- All permission decisions expressed as roles (owner, manager, watchmaker, front-desk, kiosk)
- No hardcoded user IDs anywhere in the auth layer

### SPEC-007 — Shop Floor drag-drop

Expected result: PASS all five tests. Vianna and staff use this without ownership involvement.

Watch items:
- Station configuration must be data-driven (any manager can add/reconfigure stations)
- Drag-and-drop behavior must be documented for training new staff

### SPEC-008+ — Chain of custody + IP cams

Expected result: LIKELY PASS but requires attention.

Watch items:
- IP camera review workflow must have clear escalation criteria (not "Michael reviews everything")
- Custody discrepancy resolution must have documented policies
- Access to camera footage must be role-based and audited

---

## Alignment with existing decisions

This test operationalizes the exit thesis referenced in:
- D-001 (rebuild kickoff — implicit transferability goal)
- D-018 (question snapshot discipline — documents Michael's judgment as it's applied)
- D-020 (silent failure banned — no undocumented "Michael notices" quality checks)
- D-022 (watch assembly state model — auditable, not tribal)
- RolliCurator concept (mechanism for capturing Mike Michaels' expertise)
- Authenticator V1 scope (assistive tool, staff-independent decisions)
- ROLLICLOCK-TO-RS-CONTRACT.md (documented integration contract, not tribal knowledge)

Every future decision should reference this test in its rationale.

---

## The strategic point, restated

The 11-app ecosystem, the canonical data model, the AI/data modules, the audit log framework, the two-stage intake pattern — none of it matters if the business still requires Michael Hui and Michael Michaels to function at year 5.

The Transferability Test is the design gate that ensures every technical decision serves the exit. Pass the test and Rolligroup becomes acquirable at platform multiples. Fail the test and you've built a very sophisticated version of a lifestyle business.

The 7-month sprint to 1/31/2027 isn't primarily about shipping software. It's primarily about transferring 25 years of Michael Hui's operational judgment and Mike Michaels' technical judgment into systems that persist beyond either individual.

Every SPEC weekend is a knowledge transfer session with software as the delivery format. The Transferability Test enforces that framing.

---

_End of Transferability Test doctrine. Apply to every SPEC before it locks._
