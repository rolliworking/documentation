# Build — Documents Tab + Inspection View Page

**→ Paste into: RC chat**

---

## Surgical fix only

Touch ONLY the files needed for this specific change. Don't refactor unrelated code. Don't add "while we're here" improvements. Minimal blast radius.

## Goal

Wire up the Documents tab on the client portal to show the customer's approved inspections. Build a detail view to render the inspection content. Both viewable on mobile + desktop.

## Architecture

Customer's portal has access to `client.rs_customer_id` (already in your `clients` table).

RC calls RW's new endpoint to list approvals → renders list → click to view detail.

## The RW endpoints

### List endpoint (just built today)

```
POST https://pkgnrcfqrldwjibghefm.supabase.co/functions/v1/list-customer-approvals
Headers:
  Content-Type: application/json
  x-rc-service-key: 194dfbd7333af35eca763250508a697baacdba5132c1e625d977c7919c31b261
Body:
  { "rs_customer_id": "uuid" }

Response shape:
{
  "ok": true,
  "approvals": [
    {
      "approval_id": "uuid",
      "estimate_number": "E26041",
      "is_group": false,
      "watch_count": 1,
      "watches": [
        { "brand": "Rolex", "model": "Submariner", "reference": "116610LN" }
      ],
      "status": "approved",
      "approved_at": "2026-05-17T19:34:00Z",
      "approved_by_name": "Brian Poulson",
      "total_approved": 1170,
      "items_approved_count": 2,
      "items_declined_count": 1
    }
  ]
}
```

### Detail endpoint (already exists)

```
POST https://pkgnrcfqrldwjibghefm.supabase.co/functions/v1/get-inspection-approval
Headers:
  Content-Type: application/json
Body:
  { "approvalId": "uuid" }

Returns: { approval, inspection, waiver, clientQuestions }
```

Auth: `verify_jwt = false`, public. No service key needed (UUID is the secret).

## Build

### 1. Store `RC_SERVICE_KEY` secret

Add new secret `RC_SERVICE_KEY` with value: `194dfbd7333af35eca763250508a697baacdba5132c1e625d977c7919c31b261`

Use this in the helper that calls `list-customer-approvals`.

### 2. New service helper `src/services/inspectionApprovals.ts`

```ts
export async function fetchCustomerApprovals(rsCustomerId: string) {
  // POST to list-customer-approvals with x-rc-service-key header
  // Return the approvals array
}

export async function fetchInspectionDetail(approvalId: string) {
  // POST to get-inspection-approval
  // Return the full data shape
}
```

### 3. Documents tab content

In the existing portal Documents section (placeholder today):

**List view component:**
- Fetch `fetchCustomerApprovals(client.rs_customer_id)` via React Query
- If `client.rs_customer_id` is null → show empty state "No documents yet"
- If array is empty → "No inspections approved yet"
- Otherwise render list of approval cards

**Each card displays:**
```
┌─────────────────────────────────────────────────┐
│  Inspection Report                              │
│  Rolex Submariner — Ref 116610LN                │
│  Estimate #E26041 · Approved May 17, 2026       │
│                                                 │
│  ✓ 2 items approved · $1,170                    │
│  ✗ 1 item declined                              │
│                                                 │
│  [Status pill: Approved]              [ View → ]│
└─────────────────────────────────────────────────┘
```

For group approvals (`is_group: true`):
- Show "Multi-watch approval: [watch_count] watches"
- List watches inline: "Rolex Submariner, Omega Speedmaster, Rolex GMT-Master II"

**Status pill:**
- Green "Approved" if `status === 'approved'`
- Red "Declined" if `status === 'declined'`
- Amber "Mixed" if approved AND declined items exist

### 4. New route `/client/:token/inspection/:approval_id`

Add a new route in the portal:
- Same token-based auth as existing portal routes (uses `portalClient(token)`)
- Validates `current_client_token()` like other portal routes
- Renders `InspectionViewPage` component

### 5. InspectionViewPage component

Fetches `fetchInspectionDetail(approval_id)` and renders:

```
─────────────────────────────────────────────
  Inspection Report                                     [ Print ]
  Rolex Submariner — Reference 116610LN
  Estimate #E26041
─────────────────────────────────────────────

  Plain English Summary:
  You approved 2 services totaling $1,170 on May 17, 2026.

─────────────────────────────────────────────

  Items Approved:
  ✓ Movement Service                       $850
  ✓ Dial Restoration                       $320
                                          ─────
                              Total:    $1,170

  Items Declined:
  ✗ Bracelet Polish                        $180

─────────────────────────────────────────────

  Your Notes:
  "Please handle with care"

─────────────────────────────────────────────

  Signed:    Brian Poulson
  Date:      May 17, 2026 at 2:34 PM
  Terms:     https://rolliworks.com/agreement
  
  Signed from IP 192.168.1.1
─────────────────────────────────────────────
```

Use existing portal palette (cream, serif typography, emerald green accents).

### 6. Plain English summary line

At top of detail page, generate a one-line summary:
```ts
function plainEnglishSummary(approval): string {
  const approvedCount = approvalItems.filter(i => i.choice === 'yes').length;
  const totalApproved = sum(approvedItems.price);
  const dateStr = formatDate(approved_at);
  return `You approved ${approvedCount} services totaling $${totalApproved.toFixed(2)} on ${dateStr}.`;
}
```

### 7. Print button

Simple `window.print()` call. Add `inspection-print.css`:
```css
@media print {
  /* Hide nav, sidebar, header */
  .portal-sidebar, .portal-header, .no-print { display: none !important; }
  
  /* Letter and A4 friendly */
  @page { margin: 0.5in; }
  
  /* Keep colors */
  * { -webkit-print-color-adjust: exact; print-color-adjust: exact; }
  
  /* Full width content */
  .portal-content { padding: 0; max-width: 100%; }
}
```

Browser's print dialog handles Save-as-PDF natively.

### 8. Mobile + desktop both work

- Mobile: existing tab navigation → "Documents" tab shows list → tap a row → navigate to detail
- Desktop: existing sidebar → "Documents" → list shows in right panel → click → detail in right panel

Same components, just different layout containers.

## Out of scope

- AI-generated summary beyond the simple formula above
- PDF generation (browser print is enough)
- Snapshot-on-submit (RW data is source of truth)
- "Email me a copy" button (RW is auto-emailing on submit, separate work)
- Edit/withdraw approval (not allowed per RW design)

## Acceptance

1. Documents tab on portal mobile shows list of approvals
2. Documents section on portal desktop shows same list
3. Empty state when no approvals
4. Click card → opens detail page at `/client/<token>/inspection/<approval_id>`
5. Detail page shows plain English summary + approved/declined items + signature
6. Print button works, output is clean
7. Mobile and desktop both functional
8. No regression on existing Messages/Progress sections

## Test on real customer

After deploy, test on a customer with a real approval. If none of the test accounts have one, ask RW to find a customer with non-pending approvals or sign up Brian for a test inspection.

## Tell me when done

Report back with:
- Files touched
- Any tech debt flagged
- Any concerns with the data shape from RW

---

End of prompt.
