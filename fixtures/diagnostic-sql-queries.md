---
silo: 7
date: 2026-05-14
status: working queries used during silo 7 investigations
last_validated: 2026-05-14
---

# Diagnostic SQL queries from silo 7

Queries that produced useful answers during this silo. Each one is annotated with what it was used for.

## Verify station_id migration ran

Used after B1 apply to confirm Lovable's migration file actually executed.

```sql
SELECT table_name, column_name, data_type, is_nullable
FROM information_schema.columns
WHERE table_name IN ('job_components', 'job_component_logs')
  AND column_name = 'station_id';
```

Expected result: 2 rows, both `text`, both `YES` nullable.
Got: 2 rows. Migration confirmed.

## Check current role permissions

Used to confirm manager access to shop floor was already granted (and that the "managers can't access" symptom was a non-bug).

```sql
SELECT * FROM public.role_permissions WHERE role = 'manager' ORDER BY permission_key;
SELECT * FROM public.role_permissions WHERE role = 'owner' ORDER BY permission_key;
```

Diff revealed: manager has 25 permissions, owner has 28. The 4 owner-only keys are `data.export`, `users.manage`, `users.permissions`, `users.view`. Manager has `jobs.view` (which is what gates `/shop-floor`).

## Find a user's role

Generic. Useful when "user X says they can't access Y" — first thing to check is whether their `user_roles.role` is what they think it is.

```sql
SELECT
  u.id as user_id,
  u.email,
  ur.role,
  ur.created_at as role_created_at
FROM auth.users u
LEFT JOIN public.user_roles ur ON ur.user_id = u.id
WHERE u.email = 'user@example.com';
```

Possible outcomes:
- Row with expected role → access bug is elsewhere
- Row with unexpected role → update the role
- Row with NULL role (no user_roles entry) → user defaults to `staff` in code but `useUserPermissions` returns empty, blocking access entirely. Add a user_roles row.
- Zero rows → wrong email or wrong DB

## SO 501694 forensic query

Used to confirm the duplicate-line bug pattern and identify the three write batches.

```sql
SELECT id, sort_order, entered_part_number, qty_ordered, unit_price, extended_price, created_at
FROM public.so_lines
WHERE so_id = (SELECT id FROM public.sales_orders WHERE so_number = '501694')
ORDER BY created_at, sort_order;
```

Look for:
- Tight time clusters (multiple rows with sub-second created_at differences = batch insert)
- Identical (part_number, qty, unit_price) tuples across rows = duplicates
- Same parts appearing in multiple time clusters = re-insertion across batches

## Verify station_id writes are flowing from drag

Used after B2 to confirm `commitDrop` was correctly writing `station_id`. Pre-fix this returned zero rows.

```sql
SELECT id, component_type, status, department, station_id, assignee_name, status_changed_at
FROM public.job_components
WHERE station_id IS NOT NULL
ORDER BY status_changed_at DESC
LIMIT 5;
```

After B2 fix: returns the rows that were just dragged, with `station_id` populated. Zero rows = drag handler not writing station_id (either commitDrop or handleCommit broken).

## Check for orphaned Performance Reports

Not run in silo 7 but recommended check before adding any unique constraint on `performance_verification_reports.sales_order_id`.

```sql
SELECT sales_order_id, COUNT(*) AS report_count
FROM public.performance_verification_reports
WHERE sales_order_id IS NOT NULL
GROUP BY sales_order_id
HAVING COUNT(*) > 1
ORDER BY COUNT(*) DESC;
```

Zero rows = safe to add unique constraint. Any rows = orphan reports exist, need to decide retention before adding constraint.

## EST-24392 component state and history

Used during Job History Lookup investigation as concrete sample data.

```sql
-- Current state
SELECT * FROM public.job_components
WHERE job_id = (SELECT id FROM public.jobs WHERE estimate_number = '24392');

-- Activity history
SELECT created_at, action, from_status, to_status, department, station_id, notes, changed_by
FROM public.job_component_logs
WHERE job_id = (SELECT id FROM public.jobs WHERE estimate_number = '24392')
ORDER BY created_at DESC;
```
