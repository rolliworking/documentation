# Cursor — Generate README.md for All Three Rolliworks Repos

**→ Paste into: Cursor chat**

---

## Context

We're establishing documentation discipline across the Rolliworks codebase. You have three repos locally:

- **RS (RolliSuite)** — `~/rolliworks/rollisuite` — ERP at rollisuite.com
- **RW (RolliWorking)** — `~/rolliworks/rolliworking` — workshop at rolliworking.lovable.app  
- **RC (RolliConnect)** — `~/rolliworks/rollicrm` — CRM + portal at my.rolliworks.com

All three are React + TypeScript + Vite + Tailwind + Supabase, deployed via Lovable. Each has its own Supabase project.

## Your task

Generate a `README.md` for each repo by READING its actual code (don't fabricate). For each repo:

### 1. Inspect the codebase
- Check `package.json` for dependencies and scripts
- Check `vite.config.ts` / `tsconfig.json` for build setup
- Check `supabase/config.toml` for Supabase project ID
- Check `supabase/migrations/` to understand schema
- Check `supabase/functions/` to list edge functions
- Check `src/` structure to identify main features
- Check `.env.example` or env references for required variables

### 2. Generate README.md with these sections

```markdown
# [Project Name] ([Short Code])

> One-line description of what this codebase does in the Rolliworks ecosystem.

## Quick links
- **Production:** [URL]
- **Lovable project:** [if known, otherwise note "via Lovable"]
- **Supabase project:** [project ID from config.toml]
- **Master Knowledgebase:** [link to rolliworks-docs repo when created]

## Role in the Rolliworks ecosystem
[2-3 sentences explaining what this system does and how it relates to the other two]

## Tech stack
- React [version] + Vite [version] + TypeScript
- Tailwind CSS
- Supabase (auth, postgres, edge functions, storage)
- [list any other significant deps from package.json]

## Local development

### Prerequisites
- Node.js [version from .nvmrc or package.json]
- npm or pnpm
- Supabase CLI [if needed]

### Setup
```bash
git clone [repo URL]
cd [repo name]
npm install
cp .env.example .env  # or describe env vars needed
npm run dev
```

### Environment variables
[List from .env.example or grep VITE_ in code]

## Key files & directories

```
src/
├── components/     [brief description]
├── hooks/          [brief description]
├── pages/          [brief description]
├── integrations/   [brief description]
└── lib/            [brief description]

supabase/
├── functions/      [list edge functions found]
└── migrations/     [count of migrations]
```

## Edge functions
[List ALL functions from supabase/functions/ with one-line description each]

## Database schema highlights
[List main tables from migrations — focus on the 10-15 most important]

## Key conventions
[Project-specific conventions observed in the code, e.g.:
- Auth patterns
- Naming conventions
- Token strategies
- Status enums
- Cross-system contracts]

## Deploy

### Via Lovable
[Standard Lovable deploy process]

### Edge functions only (manual)
```bash
supabase login
supabase link --project-ref [project ID]
supabase functions deploy [function-name]
```

## Cross-system integration
[Brief notes on how this system talks to the other two:
- Endpoints exposed for other systems
- Endpoints consumed from other systems
- Shared auth/tokens]

## See also
- Master Knowledgebase: [rolliworks-docs link when ready]
- [Any other relevant docs]
```

### 3. Output rules

- Read actual code; don't invent file paths or schema
- If a section is genuinely empty (e.g., no .env.example exists), note "[no env vars currently documented]" rather than fabricating
- Keep it dense and factual — no marketing fluff
- Use code blocks for commands and file structures
- Match the actual project state TODAY

### 4. Order of work

1. Generate `~/rolliworks/rollicrm/README.md` first (RC is most active)
2. Then `~/rolliworks/rolliworking/README.md`
3. Then `~/rolliworks/rollisuite/README.md`

Show me each one as you complete it. Don't commit yet — let me review.

After I approve each, I'll commit them myself.

---

End of prompt.
