# Session Log — 2026-06-07 (Sunday)

**Chat title:** Emergent rebuild + shared Supabase + RolliTime integration
**Participants:** Michael, Claude
**Duration:** Long session, multi-topic

---

## Context entering this session

- Two docs uploaded at start: `HANDOFF.md` and `PLANNING-QUESTIONS.md` (rebuild planning)
- Then: Cursor README prompt + Master Knowledgebase prompt
- Then: RolliTime integration contract
- Discussion ranged across rebuild strategy, tool selection, local AI infrastructure, documentation workflow

---

## Decisions made this session

1. **Path A confirmed** — standalone PartsWiki first, full rebuild decision deferred until documentation prerequisites exist. Documented in earlier planning docs; reinforced here.
2. **Tool stack defined** for the rebuild pipeline:
   - Existing apps + SQL + docs repo = source of truth
   - Lovable = product memory helper only
   - Local Ollama (Qwen primary) = context refinery
   - gbrain (local MCP) = persistent librarian
   - Opus = master planner
   - Cursor (interactive) = supervised module builder
   - Cursor Cloud Agents = bounded maintenance only (not new builds)
   - Emergent = module rebuilder once spec is stable
3. **gstack adopted selectively** — methodology only at first (`/office-hours`, `/plan-ceo-review`, `/retro`), full install via Cursor when ready.
4. **Ruflo rejected** — solves a problem we don't have; amplifies drift in our context.
5. **Local 112GB VRAM cluster role narrowed** — Qwen for documentation drafting, gbrain for persistent indexing. NOT for code generation, NOT for adversarial review of Opus.
6. **Hardware topology confirmed** — Corsair (96GB VRAM) as Ollama server; Legion as Ollama client; repos cloned on Legion (need to clone to Corsair for refinery work).
7. **RS confirmed as source of truth for caliber and reference-model data** — for every Rolliworks app (RW, RolliTime, future modules). Today's RS+RW drift must be reconciled before rebuild.
8. **RolliTime contract accepted as current/authoritative** — saved as `ROLLITIME-INTEGRATION.md`.
9. **Format preference established** — shorter Claude responses, fewer questions per turn (max 1-2).

---

## Artifacts produced this session

| File | Purpose |
|---|---|
| `DAILY-WORKFLOW-MAP.md` | Template for documenting how Vianna, Michael, Watchmaker Supervisor, and the Front Desk Kiosk actually use the apps |
| `FUTURE-INTEGRATIONS.md` | Living list of integrations (RolliTime, M3KE chatbot, Jarvis, watch tools, authenticator) with reserved schema slots |
| `ROLLITIME-INTEGRATION.md` | RolliTime's contract with RS, RW, and the authenticator app (saved as-is from user) |
| `REBUILD-PREREQUISITES.md` | The 7 items that must be true before rebuild starts |
| `SESSION-LOG-2026-06-07.md` | This file |
| `DECISIONS-REGISTRY.md` | Ongoing log of all architectural decisions across chats |

---

## Open questions left unresolved

1. **RC3 prerequisites** — Claude Project loaded, `.cursorrules` in repo, gstack installed, one-line goal pinned. Status: not done yet. User to execute.
2. **SQL query to compare RS vs. RW reference libraries** — pending user request (deferred end of session).
3. **Shop work orders + QBO bill-matching flow** — user mentioned, conversation pivoted to RolliTime before this was captured. Worth resurfacing next session.
4. **One-line goal anchor** — referenced in templates as a placeholder but not yet written. User to write.
5. **Photo library access control** (RolliTime §6.4) — open.
6. **Full-resolution stills vs. video frames** for authenticator app quality (RolliTime §5) — open.

---

## Things flagged but not acted on

- Rebuild timeline (4-6 months quoted to user) — Claude argued 6-8 weeks for immediate value (PartsWiki, M3KE unblock, contract fixes, docs); full shared-DB migration adds 2-3 months on top.
- Gemini doc proposing local LLM cluster as adversarial reviewer — rejected as fitting a different operator profile.
- Disciplines file content + `.cursorrules` content — offered to draft, user has not yet asked for it.

---

## Recommended next session opening

User pastes into next chat (or into the Claude Project knowledge):
- `REBUILD-PREREQUISITES.md`
- `FUTURE-INTEGRATIONS.md`
- `ROLLITIME-INTEGRATION.md`
- `DECISIONS-REGISTRY.md`
- This session log
- Plus the prior `HANDOFF.md` + `PLANNING-QUESTIONS.md`

First question for next session: status of RC3 prerequisites (done or not), then resume from there.

---

_End of session log. Save to docs repo._
