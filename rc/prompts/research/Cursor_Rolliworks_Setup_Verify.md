# Cursor — Verify Rolliworks Workflow Setup

**→ Paste into: Cursor chat**

---

## Context

I'm building three connected codebases for Rolliworks Inc (high-end watch repair business):

- **RS (RolliSuite)** — ERP at rollisuite.com, Supabase project `djbjwcoddddywkgljuja`
- **RW (RolliWorking)** — workshop/job management, Supabase project `pkgnrcfqrldwjibghefm`
- **RC (RolliConnect)** — CRM + portal at my.rolliworks.com, Supabase project `ibwrjsmuvrqtokoqpogu`

Each lives in Lovable (Claude-powered code generation). I think I've already connected them to Cursor via GitHub sync, but I want to verify the setup and identify any gaps.

## Verify the following

### 1. Repository access
- Do you have access to all three project repositories?
- If yes, list them and confirm you can browse the code
- If no, what's the connection state? (GitHub auth? Repo cloning? Permissions?)

### 2. Supabase project access
- Can you connect to the three Supabase projects via CLI?
- Do you have the project URLs and connection strings configured?
- Verify by listing tables in one of them (e.g., RC's `conversations` table)

### 3. Deploy pipeline
- If I edit code in Cursor and push to GitHub, does Lovable automatically pull and deploy?
- Or do I need to run `supabase functions deploy` manually?
- What's the safest deploy path that doesn't break Lovable's understanding of the codebase?

### 4. Branch strategy
- Each Lovable project has `main` and probably `test` (or `staging`) branches
- For RC specifically, there's a "Test → Live" merge workflow today
- Can Cursor work on the right branch? What's the recommendation for safe iteration?

### 5. Tool capabilities for my workflow
- Edge function debugging (Supabase logs)
- Multi-file refactors across the codebase
- Running migrations
- Type checking against the Supabase schema

## My typical use cases

I'd use Cursor for:
- **Surgical bug fixes** I can pinpoint (e.g., "team_users lookup needs `ilike` for case-insensitive")
- **Multi-file refactors** Lovable struggles with
- **Verifying changes Lovable made** by reading the actual code
- **Backup deploy path** when Lovable is having issues

I'd use Lovable for:
- New feature builds
- Database schema designs
- Anything UI-heavy

I'd use Claude (in a separate chat) for:
- Architecture decisions
- Cross-system coordination
- Drafting Lovable build prompts

## Specific verification task

To prove the setup works end-to-end, can you:

1. Open the RC repository
2. Find the file `supabase/functions/inbound-email-handler/index.ts`
3. Show me lines 195-210 (the recent reply token routing logic)
4. Confirm whether the recent fixes are in that file:
   - `extractToken` reads ToFull + CcFull + BccFull
   - Staff sender detection (team_users or @rolliworks.com domain)
   - Loop prevention for `*@reply.rolliworks.com` senders

If yes → setup is working and recent Lovable deploys are accessible to you.
If no → either the code isn't synced OR the fix didn't actually deploy.

## What I want from you

1. Status report on each verification step
2. Concrete walkthrough of any gaps to close
3. The verification check on the inbound-email-handler file
4. A simple "this is what we should do next" recommendation

Don't apply any code changes yet. Just verify and report.

---

End of prompt.
