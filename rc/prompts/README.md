# Prompts Library

Reusable prompts for AI build tools (Lovable, Cursor, Claude). Organized by type.

## Conventions

- Every build/research prompt is self-contained and labeled with its target tool/chat
- "Surgical fix only" is standard on build prompts to prevent scope creep
- Research prompts say "no code changes, investigate only"
- Diagnostics say "diagnose, don't fix yet"

## builds/
Build prompts that produce code/features:
- `Cursor_Internal_Notes_Build.md` — internal notes (ABANDONED, kept for reference)
- `Cursor_Generate_READMEs.md` — generate READMEs for all 3 repos
- `RC_Reply_Token_Build.md` — per-customer reply token
- `RC_Inbox_Restructure_Build.md` — categories + labels + triggers + unread
- `RC_Documents_Tab_Build.md` — Documents tab
- `RC_Get_Portal_Link_Endpoint.md` — cross-system portal URL endpoint
- `RW_Band_Flow_Split.md` — band vs watch customer flow
- `RW_Auto_Email_On_Approval.md` — auto-email on inspection approval
- `RS_Push_To_RC_Build_Prompt.md` — RS pushing data to RC
- `RS_Timeline_Status_Endpoint.md` — timeline status endpoint

## research/
Investigation prompts (no code):
- `RS_Process_Flow_Research.md` — RS process flow + shipping states
- `RS_Booking_Link_Research.md` — expose Send Booking Link to RC
- `RS_Job_Status_Research.md` — job status data
- `RC_Reply_Token_Research.md` — token system investigation
- `RC_Inbox_Restructure_Research.md` — inbox categories investigation
- `RC_Inbound_Email_Research.md` — inbound email handler
- `RC_Email_Templates_Research.md` — email templates
- `RW_Inspection_Flow_Research.md` — inspection flow
- `Cursor_Rolliworks_Setup_Verify.md` — verify Cursor setup across repos

## diagnostics/
Debugging prompts:
- `RC_Reply_Token_Diagnostic.md` — reply token routing failure
- `RC_Vienna_No_Messages_Diagnostic.md` — empty inbox
- `RS_Vienna_Invite_Diagnostic.md` — Send Portal Invite 403
- `RC_Move_To_Quoted_Diagnostic.md` — Move to Quoted button
- `RC_Trigger_Template_Diagnostic.md` — trigger template
