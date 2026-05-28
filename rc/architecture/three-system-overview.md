# Three-System Architecture Overview

**Last updated:** May 27, 2026

Rolliworks runs on three connected applications, each with its own Lovable codebase and Supabase project. This document explains how they fit together.

---

## The systems

### RS — RolliSuite (ERP)
- **Project:** `djbjwcoddddywkgljuja`
- **Production:** rollisuite.com
- **Owns:** Customer records (source of truth for customer identity), estimates, intake at the front desk, shipping/pickup states, the master process flow
- **Key role:** Where a job is born. Staff create estimates, capture customer info, scan items in.

### RW — RolliWorking (Workshop)
- **Project:** `pkgnrcfqrldwjibghefm`
- **Production:** rolliworking.lovable.app
- **Owns:** Job lifecycle on the shop floor, workshop status (`jobs.status`), flow type (`jobs.flow`), inspections, component lookup
- **Key role:** Where the actual repair work happens and status advances.

### RC — RolliConnect (CRM + Portal)
- **Project:** `ibwrjsmuvrqtokoqpogu`
- **Production:** my.rolliworks.com
- **Owns:** Customer conversations, the customer-facing portal, email routing (in + out), customer-facing status display
- **Key role:** Where customers and staff communicate; the customer experience layer.

---

## High-level data flow

```
                    ┌─────────────────┐
   Wix intake ─────▶│       RC        │  Customer submits inquiry
                    │  (portal-intake │  → conversation created
                    │   -router)      │
                    └────────┬────────┘
                             │
       Customer + staff      │  Staff qualify, discuss
       conversation          ▼
                    ┌─────────────────┐
                    │       RS        │  Staff create estimate,
                    │ (ClientWatch    │  capture customer + item details,
                    │  EntryPage)     │  scan estimate barcode
                    └────────┬────────┘
                             │
       Caret-delimited       │  sendIntakeToRolliworking()
       intake push           ▼
                    ┌─────────────────┐
                    │       RW        │  Job created with flow type,
                    │ (rollisuite-    │  workshop work begins,
                    │  intake)        │  status advances on shop floor
                    └────────┬────────┘
                             │
       Status pulled         │  rc-timeline-status / job-status
       for display           ▼
                    ┌─────────────────┐
                    │       RC        │  Customer sees progress
                    │  (portal)       │  in their portal
                    └─────────────────┘
```

---

## Inter-system contracts

### RS → RW: Intake push
- **Mechanism:** Direct browser `fetch` from RS `sendIntakeToRolliworking()`
- **Endpoint:** `https://pkgnrcfqrldwjibghefm.supabase.co/functions/v1/rollisuite-intake` (on RW)
- **Format:** Single caret-delimited (`^`) string, 18 positional fields
- **Auth:** NONE currently (`Content-Type: text/plain`, no headers) — KNOWN SECURITY GAP
- **Fields:** full_name, email, phone, part_number, date, brand, model, estimate_number, bracelet_model, service_codes, received_bracelet, received_case, dept_w, dept_b, dept_p, dept_pm, bracelet_description, received_components
- **Known gaps:** service_type not forwarded; date always = today (ignores user input); no stable customer ID; serial_number/size/material/notes not sent

### RW → RC: Status display
- **Mechanism:** RC calls RW/RS endpoint to get customer-facing status
- **Endpoint:** `rc-timeline-status` (on RC, commit `98ce8a3` introduced sectioned DTO v2 with flow-aware band/split timeline)
- **Returns:** current_step, next_step, completed_steps, flow_type, estimate_number

### RS → RC: Portal invite & inspection approval
- **Endpoints:** `rc-portal-invite` (on RS), `get-rc-portal-link` (on RC)
- **Auth:** `RC_SERVICE_KEY` shared secret
- **get-rc-portal-link** looks up `clients` by `rs_customer_id` (if provided) else `email` (case-insensitive), returns `https://my.rolliworks.com/client/<token>/c/<conv_id>`

---

## Flow type determination (band vs watch)

- **Source:** `jobs.flow` column on RW, set by `rollisuite-intake`
- **Values:**
  - `STANDARD_FLOW` — watch only
  - `SPLIT_FLOW` — watch + band
  - `BAND_ONLY_FLOW` — band only
- **Fallback when null:** derive from `dept_w` (true → watch) + `dept_b`/`dept_p`/`dept_pm` (band)
- **Used by:** BandKanban on shop floor; customer-facing timeline (band flow split work)

---

## Customer identity (the cross-system challenge)

- **RS** is the source of truth for customer identity (RolliSuite customer UUID)
- **RC** stores `clients.rs_customer_id` to link to RS
- **RW** has NO `rs_customer_id` on its customers table — email is the reliable lookup key there
- This mismatch causes lookup failures (see inspection approval URL bug in known-issues.md)

This is an active architecture concern. A dedicated effort to stabilize customer ID linkage across all three systems is planned.

---

## Strategic framing

Insight from development sessions: **RC is becoming the "OS for Rolliworks"** — not just a portal, but the central customer experience layer. Email has historically been the silo where customer communication gets lost; RC's goal is to dissolve that silo by routing all customer communication through one system.

The customer experience flywheel: portal value → adoption → habit → stickiness → retention.
