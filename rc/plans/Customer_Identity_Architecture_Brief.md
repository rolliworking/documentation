# Briefing — Customer Identity Across Rolliworks Systems

## Context

Rolliworks Inc is a high-end watch repair business. Three Lovable codebases operate together:

- **RS (RolliSuite)** — ERP at rollisuite.com, Supabase ref `djbjwcoddddywkgljuja`
- **RW (RolliWorking)** — workshop/job management at rolliworking.lovable.app, Supabase ref `pkgnrcfqrldwjibghefm`
- **RC (RolliConnect)** — customer CRM + portal at my.rolliworks.com, Supabase ref `ibwrjsmuvrqtokoqpogu`

The intended architecture per earlier decisions:
- RS owns customer identity (RS is the authoritative source)
- Wix intake → RS creates customer → RS pushes to RC (creates portal access)
- Conversations in RC link to jobs in RW

## The problem I'm trying to solve

Each system has its own `customers` table with its own UUID primary key. There is no shared customer ID across the three systems. We discovered this tonight when:

1. RC built a feature to list a customer's inspection approvals
2. RC has `rs_customer_id` stored on its `clients` table (linked to RS)
3. RC's endpoint calls RW asking for approvals with body `{rs_customer_id: <uuid>}`
4. RW reported: "Zero matches. RW's customers.id is RW-local, not synced with RS."
5. Confirmed: 4 RC customers, NONE existed in RW's customers table

## What I know today

### RS (RolliSuite)
- `customers` table with `id` UUID primary key
- This is the "source of truth" customer record
- Wix sends new intake to RS first
- RS pushes customer creation to RC via webhook
- RC stores the RS UUID as `clients.rs_customer_id`

### RW (RolliWorking)
- `customers` table with its own UUID primary key
- Created independently when jobs flow through RW
- NO link back to RS today
- NO `rs_customer_id` column on RW's customers table
- Customer email is the only consistent identifier across systems

### RC (RolliConnect)
- `clients` table for customers
- Has `rs_customer_id` column linking to RS
- Has NO link to RW customers
- 9,906 RS customers, ~300 of them have RC presence

### Data flow today
```
Wix intake → RS (creates customers row) → RC (creates clients row with rs_customer_id)
                                          → Eventually RW (creates customers row, no link back)
```

RW is the orphan in the identity chain. It receives jobs and creates customer records locally but never gets the RS UUID.

## Why this matters

Cross-system features need to lookup customers across systems:

1. RC's Documents tab needs to show a customer's approved inspections (lives in RW)
2. RC's status integration pulls from RW (already works via different lookup)
3. RS's customer detail page needs to know if customer has approvals in RW
4. Future: any feature where customer X has data in MORE than one system

Email is the only common identifier today. But emails change. Customers create multiple accounts. Emails are typed inconsistently.

## What I want from you

### Phase 1: Understand the current state
- What's the FULL picture of customer identity across all three systems?
- Where does customer data flow today?
- What lookups happen between systems currently?

### Phase 2: Diagnose the right architecture
- Should RW have an `rs_customer_id` column on its customers table?
- Or should RC's lookups use email instead of UUID?
- Or should we build a customer identity service that maps across systems?
- What are the tradeoffs?

### Phase 3: Migration plan
- For the 350+ existing approvals in RW that have no RS link: how do we backfill?
- For new approvals going forward: how should the link be established at creation time?
- What's the simplest path to "all systems agree on who the customer is"?

### Phase 4: Specific solutions for active features
- The list-customer-approvals endpoint on RW needs to work TONIGHT
- Short-term fix: lookup by email (RW reports they can do this case-insensitively)
- Long-term: RW gets `rs_customer_id` populated

## Specific questions

1. **Email as primary lookup key** — RW says they have customers with names but inconsistent emails. Some customers might have no email. Some have multiple emails. Is email reliable enough?

2. **Backfilling RW with rs_customer_id** — RW has historical data going back years. RS has 9,906 customers. How do we match the two with confidence? Exact email match? Name + email? Manual review?

3. **The dependency chain** — Today RS pushes to RC. Should RS ALSO push to RW? Should RW ask RS on every new customer creation? Where's the right insertion point?

4. **What if customer exists in RW but not RS** — Some customers may have come into RW through a path that bypassed RS (legacy data, internal techs creating records). How do we reconcile?

5. **Going forward** — When a new customer is created in any of the three systems, what's the right flow to ensure all three are linked?

## My constraints

- Surgical fixes only — don't rewrite the architecture in one shot
- Working with Lovable (Claude-powered code generation) on each codebase
- Two-person team, so any solution needs to be maintainable
- $1.5M annual revenue, ~17 service requests/day, jobs to $30K+
- 5-7 year horizon to sale, founder-dependency is real

## What I want from you specifically

1. Diagnosis: what's the right architecture for customer identity across RS/RW/RC?
2. Trade-offs: what's the cheapest fix vs the right long-term fix?
3. Migration plan: how to get from today's broken state to working state without breaking things
4. Phase 1 specific fix: what's the prompt I send to RW to make list-customer-approvals work tonight?

Don't write code. Walk through architecture, options, tradeoffs, recommendations. I'll come back with prompts after I understand.
