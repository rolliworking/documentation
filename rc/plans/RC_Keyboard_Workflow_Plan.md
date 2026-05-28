# Plan — Keyboard Workflow + Label Filter ("Answered" Folder + Arrow Keys + Delete + Label Filter)

**Captured:** Sunday May 17, 2026 (end of long sprint)
**Target build:** This week (after current backlog clears)
**Estimated effort:** ~4-5 hours

## The vision

Full keyboard-driven inbox workflow + label-based filtering. Staff blast through replies and can focus on just their assigned work.

```
1. Filter to "Mike" label → only Mike's conversations show
2. ↓ arrow → next conversation opens in right panel
3. Read it
4. Reply (mouse to compose, then back to keyboard)
5. Delete → moves to Answered, off active view
6. ↓ arrow → next conversation opens
7. Repeat
```

Four components:
- **Answered folder** — destination for handled conversations
- **Delete key shortcut** — move to Answered with one keystroke
- **Arrow key navigation** — move up/down the list with auto-open
- **Label filter dropdown** — show only conversations with a specific label

## Why this matters

Current workflow gaps:
- Vienna and Mike see ALL conversations mixed together
- No way to focus on just their assigned work
- Replying requires lots of mouse work
- No fast way to clear handled conversations

Combined improvements:
- Filter to your label → see only your work
- Keyboard navigate through them
- Delete after replying
- Active view stays clean per-person

## Updated state machine (unchanged)

```
[REQUESTS]
    │ (trigger template sent)
    ▼
[QUOTED]
    │ (client replies)
    ▼
[CONVERSATIONS]
    │ (staff hits Delete or right-click → Answered)
    ▼
[ANSWERED]
    │ (manual archive)
    ▼
[ARCHIVED]

Universal rule unchanged: ANY client reply from ANY category → CONVERSATIONS
```

## Feature breakdown

### Feature A: Answered folder

**Schema (~10 min):**
```sql
alter table conversations 
  drop constraint conversations_category_check;
alter table conversations 
  add constraint conversations_category_check 
  check (category in ('conversations','requests','quoted','answered','archived'));
```

**UI:**
- Add fifth tab "Answered" between Quoted and Archived in `CrmInbox.tsx`
- Add "Move to Answered" option to right-click context menu in `InboxList.tsx`
- No new DB trigger logic needed (existing universal "client reply → conversations" rule covers Answered → Conversations)

### Feature B: Delete key shortcut

**Behavior:**
- Press Delete while a conversation is selected → moves to Answered
- Show toast confirmation: "Moved to Answered"
- Does NOT fire when user is typing in composer, search, or any input

**Implementation:**
```ts
useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    if (e.key !== 'Delete') return;
    
    // Don't fire when typing
    const target = e.target as HTMLElement;
    if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA' || target.isContentEditable) {
      return;
    }
    
    if (!selectedConversationId) return;
    
    moveToAnswered(selectedConversationId);
    showToast('Moved to Answered');
  };
  
  window.addEventListener('keydown', handler);
  return () => window.removeEventListener('keydown', handler);
}, [selectedConversationId]);
```

### Feature C: Arrow key navigation

**Behavior:**
- Press ↓ → selection moves to next conversation in current tab → auto-opens in right panel
- Press ↑ → selection moves to previous conversation in current tab → auto-opens in right panel
- Stays within current tab (Conversations stays in Conversations, Requests stays in Requests, etc.)
- At end of list: stops (no wrap, no jump to next tab)
- All conversations are navigable (no read/unread filtering)
- Does NOT fire when typing in composer, search, or any input

**Visual:**
- Selected conversation gets a small ring/border around the row
- Selection persists across navigation (doesn't reset on category switch)

**Implementation hint:**
```ts
useEffect(() => {
  const handler = (e: KeyboardEvent) => {
    if (e.key !== 'ArrowDown' && e.key !== 'ArrowUp') return;
    
    // Don't fire when typing
    const target = e.target as HTMLElement;
    if (target.tagName === 'INPUT' || target.tagName === 'TEXTAREA' || target.isContentEditable) {
      return;
    }
    
    e.preventDefault(); // prevent page scroll
    
    const currentIndex = filteredConversations.findIndex(c => c.id === selectedConversationId);
    if (e.key === 'ArrowDown' && currentIndex < filteredConversations.length - 1) {
      const nextConv = filteredConversations[currentIndex + 1];
      selectConversation(nextConv.id);
      // openConversation() if not already auto-opening
    }
    if (e.key === 'ArrowUp' && currentIndex > 0) {
      const prevConv = filteredConversations[currentIndex - 1];
      selectConversation(prevConv.id);
    }
  };
  
  window.addEventListener('keydown', handler);
  return () => window.removeEventListener('keydown', handler);
}, [selectedConversationId, filteredConversations]);
```

### Feature D: Label filter dropdown

**Behavior:**
- Dropdown above the inbox list (next to or near the tab buttons)
- Options: "All" (default), "🟡 Update work order", "🔴 Vienna", "🟢 Mike"
- Pick one → list filters to only conversations with that label
- "All" → clears filter, shows all conversations in current tab
- One label at a time (no multi-select)
- Filter is per-tab — switching tabs preserves the filter selection

**Visual:**
- Small dropdown with colored dots matching the label colors
- Active filter visible in the dropdown button (e.g., "🟢 Mike")
- Clear visual: when filter is active, the inbox header could show the active label
- Empty state: "No conversations with this label in [Tab Name]"

**Combines with other features:**
- Filter active → arrow keys navigate only filtered conversations
- Filter active → Delete moves filtered conversation to Answered (visible in Answered tab without filter)
- Filter is independent of category — works across all tabs

**Implementation hint:**
```ts
// In InboxList.tsx or CrmInbox.tsx
const [labelFilter, setLabelFilter] = useState<'all' | 'yellow' | 'red' | 'green'>('all');

const filteredConversations = useMemo(() => {
  const tabFiltered = conversations.filter(c => c.category === currentTab);
  if (labelFilter === 'all') return tabFiltered;
  return tabFiltered.filter(c => 
    c.labels?.some(l => l.label === labelFilter)
  );
}, [conversations, currentTab, labelFilter]);
```

## Acceptance

### Answered folder
1. New "Answered" tab visible between Quoted and Archived
2. Right-click → "Move to Answered" option works
3. Client reply from Answered → bumps to Conversations (existing universal rule)

### Delete key
1. Conversation selected → press Delete → moves to Answered + toast
2. Conversation selected → typing in composer → Delete does NOT move conversation
3. Works from any tab

### Arrow keys
1. Press ↓ → selection moves to next conversation in current tab → opens in panel
2. Press ↑ → selection moves to previous conversation → opens in panel
3. At end of list (top or bottom) → stops, doesn't wrap
4. Selection visual: ring/border around selected row
5. Typing in composer or search → arrow keys don't navigate (default text editor behavior)
6. Switching tabs → selection resets to first in new tab

### Label filter
1. Dropdown visible near tab navigation
2. Options: All, Yellow, Red, Green (with names)
3. Pick one → list shows only matching conversations
4. Pick "All" → all conversations shown
5. Filter persists across tab switches
6. Empty state when no conversations match filter
7. Filter works combined with other features (arrow nav respects filter)

## Out of scope

- Other keyboard shortcuts (Q for Quoted, A for Archive — could come later)
- Bulk selection (cmd-click multiple, then keyboard action)
- Vim-style shortcuts (j/k for nav)
- Cross-tab navigation
- Mobile equivalent (touch-only)
- Multi-label filtering (one label at a time only)
- User-extensible labels (fixed three colors)

## Build order

1. Schema change (add 'answered' to category check) — 10 min
2. Inbox tabs + right-click "Move to Answered" — 30 min
3. Delete key shortcut + toast — 30 min
4. Label filter dropdown — 45 min
5. Arrow key navigation + selection visual — 1-2 hr
6. Test the input-field exclusion carefully across all three — 30 min

Total: ~4-5 hours focused build

## Considerations for later

- More shortcuts: Q for Quoted, A for Archive, R for reply focus
- Auto-move to Answered when team sends reply (configurable)
- Vim-style navigation alternatives
- Numbered shortcuts (1-9 for first 9 conversations)

## Pickup notes

When resuming:
1. Build Answered folder first (lowest dependency)
2. Then Delete key (depends on Answered)
3. Then Arrow keys (independent but bundles naturally)
4. All three share the same "don't fire when typing" pattern — factor into a useKeyboardShortcut hook
