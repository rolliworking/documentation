# Plan — Lifetime Spend Badge on Inbox Conversations

**Captured:** Sunday May 17, 2026 (end of long sprint)
**Target build:** This week (after current backlog clears)
**Estimated effort:** ~3 hours cross-system

## The vision

Staff sees a small badge below each customer's name in the inbox showing their lifetime total spend with Rolliworks. Glanceable customer-value context for every conversation.

Format: compact dollar (e.g., "12k", "15k", "$850")

Examples:
```
Brian Poulson
12k

Vienna Smith  
3k

Mike Johnson
850
```

## Why this matters

- Staff knows at-a-glance if they're replying to a VIP or new customer
- Tone calibration: a $30k-lifetime customer deserves different attention than a first-timer
- Repeat customer recognition (Wix never showed this)
- Compounds with status badges, labels, unread count — every reply gets more context

## Format specification

- Numbers under $1,000: show as exact dollars ($850)
- Numbers $1,000+: show as "Xk" (12k, 15k, 230k)
- No badge if customer has zero spend (first-time inquiries)
- Position: below customer name, smaller font, muted color

## Data flow

```
RS customers table (with estimates joined)
    ↓ (RS edge function: GET /rc-customer-ltv?customer_id=X)
RC inbox row component
    ↓ (renders badge)
Inbox list shows lifetime value below name
```

## What needs to change

### RS side (~1 hr)

1. **New endpoint `rc-customer-ltv`**
   - GET param: `rs_customer_id`
   - Returns: `{ ok: true, lifetime_spend: 12450 }`
   - Sums approved/paid estimates for that customer
   - Auth via existing `RC_INBOUND_KEY`
   - Cache 60s (same pattern as `rc-timeline-status`)

### RC side (~2 hrs)

1. **New hook `useCustomerLTV(rs_customer_id)`**
   - Calls RS endpoint
   - Returns formatted string (12k, 850, etc.)
   - React Query for caching
2. **Inbox row component update**
   - Add LTV badge below customer name
   - Compact muted styling
   - Only render if value > 0
3. **Format helper**
   - `formatLTV(amount)` → "12k" / "$850" / "" 
   - Round to nearest hundred for >$1k
   - Strict dollar for under

## Build phases

### Phase 1: RS endpoint (~1 hr)
- Build `rc-customer-ltv` edge function
- Sum query from estimates table
- Test with known customer (Brian or similar)

### Phase 2: RC consumer (~2 hrs)
- Hook + format helper
- Inbox row badge display
- Verify across all four tabs

## Edge cases to handle

1. **Customer with no rs_customer_id linked** → no badge
2. **Customer with $0 lifetime spend** → no badge (don't show "0")
3. **Performance**: inbox loads 50+ conversations — fetch LTV in batch, not per-row
4. **Stale data**: 60s cache OK, lifetime values don't change rapidly

## Out of scope

- Tier styling (no VIP/Regular/Premium labels)
- Order count display (just dollars)
- Last 12 months spend (lifetime only)
- Edit/override LTV manually
- Customer detail page integration (just inbox for now)

## Considerations for later

- Add tier icons (💎 for $10K+, ⭐ for $5-10K)
- Customer detail page shows full breakdown (per-year, per-watch)
- Top-spender dashboard view in CRM
- LTV-based filtering in inbox

## Pickup notes

When resuming:
1. Check if RS already has an LTV computation anywhere
2. Build RS endpoint with same auth pattern as `rc-timeline-status`
3. RC consumer follows same pattern as the status badge consumer

## Related work

- Customer ID architecture decision (pending tomorrow's research)
- This depends on customer-data flow stability between RS ↔ RC
