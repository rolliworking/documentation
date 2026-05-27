---
silo: 6
discovered: 2026-05-14
status: mitigated (convention established, not enforced)
last_validated: 2026-05-14
---

# Gap — Lovable plan-mode vs apply-mode confusion

## The problem

Lovable has two modes (plan and apply). Both produce IDENTICAL "Files saved: yes / line ranges / final function as written" confirmation language. You cannot tell from Lovable's response which mode you're in.

This caused multiple "shipped!" celebrations in silo 6 for items that were actually still plan-pending.

## Why it matters

Future AI sessions and Mike both need to know what's actually deployed. Misreading plan-mode output as apply-mode output produces:
- Wrong belief about production state
- Doubled validation work later
- Lost trust in the workflow

## Mitigation (established this silo)

Mike's convention: only click the Approve button in Lovable when the prompt is framed as **"Surgical fix"** at the top. Plan-only investigations use **"Mode: Surgical investigation, NO code changes"** or **"Mode: Plan first, apply after I confirm"**.

Future AI sessions should:
1. Never assume Lovable's "Files saved" means applied
2. Verify against production: database state for SQL, code in git for source
3. Ask Mike "did you click Approve?" if uncertain

## Permanent fix

Would require Lovable's UI to differentiate plan vs apply responses textually. Not in our control.

## References

- Silo 6 chat, late-session correction where Mike clarified the Approve trigger
- Multiple silo 6 items still have unverified apply state (Plan A, drag v1, tech picker v2, dotStationFor)
