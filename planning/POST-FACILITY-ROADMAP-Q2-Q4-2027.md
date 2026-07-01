# Post-Facility Roadmap — Q2–Q4 2027

> **Status:** Planning draft — everything explicitly **OUT** of the 1/31/2027 facility cutover.
> **Companion:** `SCOPE-HOLD-1-31-2027.md`, `FUTURE-INTEGRATIONS.md`, WISHLIST.md
> **Assumption:** Facility opens ~February 2027; core ops stable by Q2 2027 before greenfield modules begin.

---

## Timeline overview

```
2027 Q1          2027 Q2              2027 Q3              2027 Q4
│ Facility       │ Stabilize +         │ AI / data           │ Platform
│ cutover        │ RT integration    │ modules v1          │ expansion
│ SPEC-001–010   │ W-40 prep         │ Authenticator       │ RolliShop
│                │ RolliCurator MVP  │ Jarvis pilot        │ PartsWiki
```

---

## Deferred modules

### 1. Authenticator V1 (AA)

| Attribute | Detail |
|-----------|--------|
| **What** | Genuine vs counterfeit verification using dial-photo corpus |
| **Language** | Python / CV per D-017 |
| **Prerequisites** | RolliTime dial library at scale; `shared.intake_photos` index; D-021 R2 pipeline |
| **Dependencies** | RT integration live; photo quality decision (ROLLITIME-INTEGRATION §5) |
| **Earliest start** | Q3 2027 |
| **Estimated duration** | 8–12 weeks to assistive V1 (staff-in-loop, not autonomous) |
| **Transferability waiver** | Mike Test — Mike Michaels is authoritative source until RolliCurator dataset sufficient |

### 2. RolliCurator wizard

| Attribute | Detail |
|-----------|--------|
| **What** | Multi-AI quiz wizard capturing validated expertise into master tables (D-012) |
| **Language** | Python per D-017 |
| **Prerequisites** | SPEC-002 canonical model live; `shared.*` reference tables with promotion_status |
| **Dependencies** | D-011 promotion workflow implemented |
| **Earliest start** | Q2 2027 (parallel with stabilization) |
| **Estimated duration** | 6–10 weeks to MVP (calibers + references domain first) |
| **Strategic value** | Transferability — captures Mike-level expertise into system |

### 3. Jarvis AI assistant

| Attribute | Detail |
|-----------|--------|
| **What** | Operations LLM agent — daily hit list, escalations, internal Q&A (W-21 seam) |
| **Language** | Python per D-017 |
| **Prerequisites** | Clean audit log; structured customer/watch data; M3KE patterns |
| **Dependencies** | RolliCurator dataset; `ai_modules.*` auth pattern (REBUILD-PREREQUISITES #3) |
| **Earliest start** | Q4 2027 |
| **Estimated duration** | 12+ weeks — scope fuzzy |
| **Note** | Explicitly OUT of facility cutover per scope hold |

### 4. PartsWiki

| Attribute | Detail |
|-----------|--------|
| **What** | Standalone parts knowledge base (D-001 Path A origin) |
| **Status** | Deferred since original 3-month PartsWiki-first plan |
| **Prerequisites** | RS parts master stable; M3KE integration for parts requests |
| **Earliest start** | Q4 2027 |
| **Estimated duration** | TBD — revisit after facility ops stable |

### 5. RolliShop full launch

| Attribute | Detail |
|-----------|--------|
| **What** | Multi-location retail expansion (Multiplication Test driver) |
| **Prerequisites** | D-023 passing SPECs; multi-tenant config data-driven; location deployment runbooks |
| **Dependencies** | Facility #1 proven; SCOPE-HOLD IN items all stable 90+ days |
| **Earliest start** | 2028 |
| **Estimated duration** | Per-location rollout — not a single sprint |

### 6. M3KE chatbot embedded in RolliSuite

| Attribute | Detail |
|-----------|--------|
| **What** | In-app escalation chatbot (M3KE v1 exists as standalone) |
| **Prerequisites** | Scoped service role auth (Q-010); RS JWT integration fixed |
| **Earliest start** | Q3 2027 |
| **Estimated duration** | 4–6 weeks after auth pattern locked |

### 7. W-40 AI warranty summary reports

| Attribute | Detail |
|-----------|--------|
| **What** | Client-facing AI report pulling test data + photos + service history |
| **Prerequisites** | `shared.timing_test_results`; D-021 photo index; RT integration |
| **Earliest start** | Q3 2027 |
| **Estimated duration** | 4–8 weeks (model selection deferred) |

### 8. W-41 Variance notifier at Receive Watch

| Attribute | Detail |
|-----------|--------|
| **What** | Inline estimate variance resolution at Stage 2 |
| **Note** | May move **IN** if SPEC-001 scope expands — currently P1 post-core intake |

### 9. W-42 Cron D-020 audit (maintenance-hourly, vault-storage-fee)

| Attribute | Detail |
|-----------|--------|
| **What** | Document and fix undocumented crons |
| **Earliest start** | Q2 2027 (ops hygiene, not blocking opening) |

---

## Integration milestones (post-facility)

| Milestone | Target | Blocker |
|-----------|--------|---------|
| RolliTime live API sync | Q2 2027 | Q-008 contracts implemented |
| Historical photo backfill index | Q2 2027 | D-021 backfill script |
| RC email reply routing | Q2 2027 | Q-007-A answer |
| Full QBO sync rewrite | Q2 2027 | PROD-FIX-001 + D-020 chunked cron |
| Chain of custody cameras live | Q2–Q3 2027 | D-015 + facility checklist F-13–F-19 |
| Reference data RolliCurator seed | Q3 2027 | D-011 promotion workflow |

---

## Resource assumptions

| Role | Q2 2027 focus | Q3–Q4 2027 focus |
|------|---------------|------------------|
| Michael Hui | Stabilize ops, approve SPECs | RolliCurator content, scope guard |
| Mike Michaels | Training, technical validation | Authenticator training data |
| Vianna | Run new intake/floor flows | RC + customer comms |
| AI engineering (Cursor/Emergent) | Bug fixes, P1 items | Greenfield Python modules |

---

## Review cadence

- **Monthly** — revisit this roadmap against actual facility opening date (SPEC-PREP-003)
- **Quarterly** — Transferability Test sample on live systems (D-023)
- **On scope creep** — point to `SCOPE-HOLD-1-31-2027.md`

---

_End of post-facility roadmap. Not committed scope — planning artifact only._
