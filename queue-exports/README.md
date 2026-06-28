# Queue Exports — Now in Dropbox

> **Snapshot location moved to Dropbox as of 2026-06-28.**
> See `DECISIONS-REGISTRY.md` D-018 (amended) for the rationale.

## Where to find current snapshots

```
G:\Dropbox\__AI\emergent-hq-human-queue\
```

This folder is on Michael's Dropbox account. Auto-syncs across all devices.

## Why the move

Word docs are binary — git history of them adds no value. Putting snapshots in Dropbox enables:
- Cross-device access without git pull/push
- Auto-sync to phone, iPad, work computer, Legion
- Answering questions on any device during downtime
- No manual copy step from repo → Dropbox

The markdown source of truth (`HUMAN-QUEUE.md` and `HUMAN-ANSWERS.md` in the rebuild-hq repo) stays in git. Only the Word doc snapshots live in Dropbox.

## How to generate a fresh snapshot (Cursor prompt)

```
Generate a new questions snapshot Word document.

Source data:
1. Read HUMAN-QUEUE.md from rebuild-hq root
2. Read every file in documentation/discovery/ and extract the "Open questions" section from each
3. Deduplicate questions that appear in both sources
4. Sort by Q-ID

Output format:
- Word document following the structure of the most recent QUESTIONS-*.docx in G:\Dropbox\__AI\emergent-hq-human-queue\
- Cover page with date and stats (total questions, source files, estimated time to answer)
- One section per question with: ID, source task, type, question text, why it matters, what was observed, best guess, default if no answer in 7 days, and a blank "YOUR ANSWER" box
- Last page repeats this regeneration prompt for next time

Save location: G:\Dropbox\__AI\emergent-hq-human-queue\QUESTIONS-YYYY-MM-DD.docx with today's date.

This file does NOT get committed to the repo. The git workflow is for HUMAN-ANSWERS.md only.
```

## How answered docs return

1. Operator opens the Word doc on any device (Dropbox auto-syncs it)
2. Types answers into the YOUR ANSWER boxes
3. Saves the Word doc (still in Dropbox, same filename)
4. Tells Cursor: "Read the answered Word doc at G:\Dropbox\__AI\emergent-hq-human-queue\QUESTIONS-YYYY-MM-DD.docx, extract the answers, and update HUMAN-ANSWERS.md in rebuild-hq accordingly. Commit and push."
5. (Optional) Move the answered doc to `G:\Dropbox\__AI\emergent-hq-human-queue\answered\` to preserve historical record

## The full loop

```
Qwen runs discovery
  ↓
Logs questions to HUMAN-QUEUE.md (in git, source of truth)
  ↓
Cadence trigger → Word doc generated in Dropbox
  ↓
Dropbox syncs to phone/iPad/work/Legion
  ↓
Operator answers on any device during downtime
  ↓
Cursor reads answered doc from Dropbox → updates HUMAN-ANSWERS.md in git
  ↓
Qwen reads HUMAN-ANSWERS.md → resumes blocked tasks
```

No step requires the operator to be at any specific device.

## Historical files

This folder (`documentation/queue-exports/` in the repo) may still contain old Word docs from before the Dropbox move. They are kept for historical reference but are no longer the active workflow surface.

## See also

- `DECISIONS-REGISTRY.md` D-018 (amended) — the discipline as a decision
- `BUILD-PROCESS.md` "Question snapshot cadence" section — workflow integration
- `QWEN.md` "Ambiguity handling" — the format Qwen uses when logging
- `HUMAN-QUEUE.md` in rebuild-hq — the live source data
- `HUMAN-ANSWERS.md` in rebuild-hq — where answers land
