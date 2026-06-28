# Queue Exports

Periodic Word document snapshots of accumulated human questions from the autonomous rebuild workflow.

## Why this folder exists

Qwen (the discovery agent) logs questions it can't answer from code alone to `HUMAN-QUEUE.md` in the rebuild-hq repo. That file accumulates over time and becomes hard to scan, especially on mobile or while away from the keyboard.

This folder holds dated Word documents that snapshot the accumulated questions in a more usable format:
- Scannable on phone or tablet
- Printable if needed
- Structured with answer boxes for each question
- Easier to work through during downtime

See `DECISIONS-REGISTRY.md` D-018 for the discipline behind this folder.

## File naming

Every snapshot follows the pattern:

```
QUESTIONS-YYYY-MM-DD.docx
```

The date is the day the snapshot was generated, not the day questions were logged.

## When new snapshots get generated

Per D-018 and BUILD-PROCESS.md, a new snapshot is generated when any of these fire:

- Every 72 hours (alongside the build digest)
- HUMAN-QUEUE.md has 10+ open questions
- Before any Opus planning session
- Before any Qwen autonomous run lasting 4+ hours

## How to regenerate (Cursor prompt)

Paste this into Cursor anytime you want a fresh snapshot:

```
Generate a new questions snapshot Word document.

Source data:
1. Read documentation/HUMAN-QUEUE.md from rebuild-hq (or root-level HUMAN-QUEUE.md if that's where it lives)
2. Read every file in documentation/discovery/ and extract the "Open questions" section from each
3. Deduplicate — questions that appear in both HUMAN-QUEUE and a discovery file count once
4. Sort by Q-ID

Output format:
- Word document following the structure of the most recent QUESTIONS-*.docx in this folder
- Cover page with date and stats (total questions, source files, estimated time to answer)
- One section per question with: ID, source task, type, question text, why it matters, what was observed, best guess, default if no answer in 7 days, and a blank "YOUR ANSWER" box
- Last page repeats this regeneration prompt for next time

Filename: documentation/queue-exports/QUESTIONS-YYYY-MM-DD.docx with today's date.

Commit message: "Generate questions snapshot YYYY-MM-DD"
```

## How to use a snapshot

1. Open the Word doc on whatever device is convenient (phone, tablet, work laptop)
2. Read questions one at a time
3. Type your answer in the "YOUR ANSWER" box beneath each
4. Save the document as you go — partial completion is fine
5. When done (or partway done), send the doc back via email, file share, OneDrive, or drop in the queue-exports folder
6. Tell Cursor: "Read the answered Word doc at <path>, extract the answers, and update HUMAN-ANSWERS.md in rebuild-hq accordingly. Commit and push."

## The full loop

```
Qwen runs discovery
  ↓
Logs questions to HUMAN-QUEUE.md
  ↓
Periodic trigger fires → Word doc generated in this folder
  ↓
Operator answers questions offline
  ↓
Cursor reads answered Word doc → updates HUMAN-ANSWERS.md
  ↓
Qwen reads HUMAN-ANSWERS.md → resumes blocked tasks
```

No step requires the operator to scroll the raw HUMAN-QUEUE.md markdown.

## Notes

- Don't edit completed Word docs after answers have been extracted to HUMAN-ANSWERS.md — they become historical record
- If a question appears in multiple snapshots (because it was logged once but never answered), that's a signal it's been open too long
- The README itself doesn't get regenerated — only the QUESTIONS-*.docx files

## See also

- `DECISIONS-REGISTRY.md` D-018 — the discipline as a decision
- `BUILD-PROCESS.md` "Question snapshot cadence" section — workflow integration
- `QWEN.md` "Ambiguity handling" — the format Qwen uses when logging
- `HUMAN-QUEUE.md` in rebuild-hq — the live source data
- `HUMAN-ANSWERS.md` in rebuild-hq — where answers land
