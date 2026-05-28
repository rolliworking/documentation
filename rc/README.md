# RolliConnect (RC)

Documentation scoped to **RolliConnect** — client CRM, team inbox, and customer portal at [my.rolliworks.com](https://my.rolliworks.com).

## Repo & infra

| Item | Value |
|------|--------|
| GitHub | `rolliworking/rollicrm-scaffold` |
| Local path | `~/rolliworks/rollicrm` |
| Active branch | `Test` |
| Supabase project | `ibwrjsmuvrqtokoqpogu` |
| Production URL | https://my.rolliworks.com |

## This folder

RC-specific silos, gaps, decisions, fixtures, and handoff notes live here. Cross-repo items (RS→RW→RC flows, beacons, shared Supabase decisions) remain in the repo root until moved or cross-linked.

| Path | Purpose |
|------|---------|
| `rc/silos/` | RC-focused silo closeouts |
| `rc/handoff/` | RC rolling handoff |
| `rc/known-gaps/` | RC-only gaps and drift |
| `rc/decisions/` | RC architecture and workflow decisions |
| `rc/fixtures/` | RC portal / inbox test fixtures |

## Related docs (repo root)

Cross-repo RC work is documented alongside RS/RW in the shared folders:

- [Silo 9](../silos/silo-9.md) — RC timeline v2, receive-watch beacon
- [Decision: RC sectioned DTO v2](../decisions/2026-05-25-rc-sectioned-dto-v2.md)
- [Decision: Shared Supabase](../decisions/2026-05-25-shared-supabase.md)
- [Fixture: beacon EST 25530](../fixtures/beacon-est-25530.md)
- [Handoff: current (all repos)](../handoff/current.md)

## Edge functions (RC)

| Function | Purpose |
|----------|---------|
| `get-rc-portal-link` | RW → RC lookup: email or `rs_customer_id` → portal URL |
| `portal-intake-router` | Wix / intake routing into conversations |
| `inbound-email-handler` | Postmark inbound replies |
| `timeline-status` | Portal progress (proxies RS `rc-timeline-status`) |
