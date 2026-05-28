# Build — `get-rc-portal-link` Endpoint + Find Test Customer

**→ Paste into: RC chat**

---

## Surgical fix only

Touch ONLY the new edge function. No other changes.

## Goal

Build a new edge function that RW can call to look up a customer's RC portal URL by email or rs_customer_id. Used in the inspection approval auto-email so customers land on RC portal instead of RW.

## Endpoint spec

**Path:** `supabase/functions/get-rc-portal-link/index.ts`
**Method:** POST
**Auth:** `verify_jwt = false`, validate `x-rc-service-key` header matches `RC_SERVICE_KEY` secret (same value already added today)

**Body — accepts either:**
```json
{ "email": "customer@example.com" }
```
OR
```json
{ "rs_customer_id": "uuid" }
```

**Response — success:**
```json
{
  "ok": true,
  "portal_url": "https://my.rolliworks.com/client/TOKEN/c/CONVERSATION_ID"
}
```

**Response — no match:**
```json
{ "ok": false, "error": "client not yet in RC" }
```

**Logic:**
1. Validate `x-rc-service-key` header
2. Lookup in `clients` table by email OR rs_customer_id (whichever provided)
3. If client found:
   - Get most recent conversation for that client
   - Construct URL: `https://my.rolliworks.com/client/${client.token}/c/${conversation.id}`
   - Return success
4. If no client OR no conversation: return `{ ok: false, error: ... }`

If a client has multiple conversations, use the most recent (most recently updated).

## Test commands

After deploy, test these:

```bash
# Should succeed (Brian Poulson email + key)
curl -X POST https://ibwrjsmuvrqtokoqpogu.functions.supabase.co/functions/v1/get-rc-portal-link \
  -H "Content-Type: application/json" \
  -H "x-rc-service-key: 194dfbd7333af35eca763250508a697baacdba5132c1e625d977c7919c31b261" \
  -d '{"email":"sunflashx@gmail.com"}'

# Should fail (no key)
curl -X POST https://ibwrjsmuvrqtokoqpogu.functions.supabase.co/functions/v1/get-rc-portal-link \
  -H "Content-Type: application/json" \
  -d '{"email":"sunflashx@gmail.com"}'

# Should return not found
curl -X POST https://ibwrjsmuvrqtokoqpogu.functions.supabase.co/functions/v1/get-rc-portal-link \
  -H "Content-Type: application/json" \
  -H "x-rc-service-key: 194dfbd7333af35eca763250508a697baacdba5132c1e625d977c7919c31b261" \
  -d '{"email":"doesnotexist@example.com"}'
```

## Also do — find a test customer for end-to-end testing

I want to test the inspection auto-email flow tonight. To do that I need a customer who:
1. Exists in RC (has portal access)
2. Has a pending inspection in RW

Query your database AND ask RW to query theirs to find a candidate. Or list 2-3 candidates with their:
- Customer name
- Email
- Whether they have a pending inspection in RW

If no candidate exists, suggest creating a test approval flow for Brian Poulson or another test customer.

## Acceptance

1. Endpoint deployed
2. Returns success for Brian Poulson (sunflashx@gmail.com)
3. Returns failure for unknown emails
4. Returns 401 without service key
5. Tell me at least one customer we can use for live end-to-end testing

---

End of prompt.
