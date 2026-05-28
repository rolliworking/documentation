# Plan — Shared Team Read State

**Captured:** Sunday May 17, 2026
**Target build:** Tomorrow morning (high priority — Vienna is hitting this NOW)
**Estimated effort:** ~1-2 hours

## The problem

Today read state is per-user. When Mike reads/replies to a conversation, Vienna still sees it as unread. This creates friction for a 2-person shared inbox.

Vienna reports seeing many unread emails that Mike has already read and replied to.

## The decision

Read state becomes **team-wide**:
- One read state per conversation (shared across all team members)
- Opening OR replying marks as read for the whole team
- Mark as Unread is also team-wide (everyone sees it unread)

## Why this is right for Rolliworks

- Mike + Vienna are 2 people running ONE shared inbox
- They both reply to customers
- "Handled by team" is the meaningful state
- Personal read tracking is friction without benefit

This matches the mental model of help@rolliworks.com — it's a team mailbox, not personal mail.

## Architecture changes

### Schema (~15 min)

Two options:

**Option A: Drop user_id from conversation_read_states**
- Make it one row per conversation
- `last_read_at` becomes team-shared
- `marked_unread_at` becomes team-shared
- Simplest

**Option B: Keep per-user table but aggregate**
- Add a new view or RPC that returns "team-wide read state"
- get_inbox_unread_counts aggregates: a conversation is unread if NO team member has read since latest client message
- More complex but preserves per-user history if needed later

**Recommendation: Option A** — cleaner, simpler. Lose per-user history but it's not used.

```sql
-- Migration
alter table conversation_read_states drop column user_id;
-- Or recreate with conversation_id as PK
```

### get_inbox_unread_counts RPC (~30 min)

Update the existing RPC to use team-wide state instead of per-user:

```sql
create or replace function get_inbox_unread_counts()
returns table(conversation_id uuid, unread_count bigint) as $$
  with msg_counts as (
    select 
      m.conversation_id, 
      count(*) as msg_unread
    from messages m
    left join conversation_read_states crs 
      on crs.conversation_id = m.conversation_id 
    where m.author_type = 'client'
      and m.created_at > coalesce(crs.last_read_at, '1970-01-01'::timestamptz)
    group by m.conversation_id
  ),
  manual_unread as (
    select conversation_id, 1 as marked
    from conversation_read_states
    where marked_unread_at is not null
      and marked_unread_at > coalesce(last_read_at, '1970-01-01'::timestamptz)
  )
  select 
    coalesce(mc.conversation_id, mu.conversation_id) as conversation_id,
    greatest(coalesce(mc.msg_unread, 0), coalesce(mu.marked, 0)) as unread_count
  from msg_counts mc
  full outer join manual_unread mu on mc.conversation_id = mu.conversation_id
  where coalesce(mc.msg_unread, 0) > 0 or coalesce(mu.marked, 0) > 0;
$$ language sql stable;
```

Note: drops the `team_user_id` parameter. The function now returns team-wide counts.

### Mark conversation as read (~15 min)

When team member opens a conversation OR sends a reply:
- Update `conversation_read_states.last_read_at = now()` for that conversation
- No user_id needed
- Everyone benefits

### Mark as Unread (~15 min)

When team member clicks Mark as Unread:
- Update `conversation_read_states.marked_unread_at = now()` for that conversation
- Everyone sees it unread

### Frontend (~30 min)

Update calls to `get_inbox_unread_counts()` to not pass user_id.
Update read/unread mutations to not pass user_id.
No major UI changes needed — the badge already shows team-wide counts will just work.

## Acceptance

1. Mike opens a conversation → Vienna's inbox no longer shows it as unread
2. Mike sends a reply → conversation marked read for the team
3. Mike clicks Mark as Unread → Vienna sees it as unread too
4. New client message arrives → both Mike and Vienna see it as unread
5. Either Mike or Vienna can open → both see it as read
6. Numeric badge shows correct shared count

## Edge cases

1. **Both staff open simultaneously** — last write wins, that's fine
2. **Existing per-user data** — drop user_id column, keep latest last_read_at per conversation (pick the most recent)
3. **Future: 3+ staff** — same model scales (team-wide regardless of headcount)

## Out of scope

- Per-user "snooze" or "remind me" features (defer)
- Showing WHO last read a conversation (could add for accountability later)
- Assignment to specific staff (separate feature)

## Migration strategy

For existing data:
1. For each conversation, find the MOST RECENT `last_read_at` across all users → that becomes the team-wide value
2. Drop user_id column
3. Make conversation_id the primary key

```sql
-- Migration script (run carefully, test on staging first)
begin;

-- Consolidate per-user state into team-wide state
create table conversation_read_states_new as
select 
  conversation_id,
  max(last_read_at) as last_read_at,
  max(marked_unread_at) as marked_unread_at
from conversation_read_states
group by conversation_id;

alter table conversation_read_states_new add primary key (conversation_id);

drop table conversation_read_states;
alter table conversation_read_states_new rename to conversation_read_states;

-- RLS: team users can read/write all rows
alter table conversation_read_states enable row level security;
create policy "team manages read state" on conversation_read_states
  for all using (auth.uid() in (select user_id from team_users));

commit;
```

## Tests

After deploy:
1. Mike opens a conversation → check Vienna's inbox shows it read
2. New customer message → both Mike and Vienna see badge
3. Either replies → both see read
4. Mike Marks as Unread → Vienna sees unread badge
5. Either opens → badge clears for both

## Priority

**HIGH** — Vienna is actively confused by current behavior. Quick fix tomorrow morning.

## Related work

- Customer ID architecture decision (separate, tomorrow)
- Multi-job display (Memorial Day weekend)
- Status notifications (this week)

## Pickup notes

When resuming:
1. Take migration script carefully — test on a backup first
2. Lovable's auto-backup before deploy is helpful here
3. Verify Vienna can actually see the fix (test with her account if possible)
