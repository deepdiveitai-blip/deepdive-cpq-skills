---
name: dd-cpq-meeting-actions
description: >-
  Turn a Zoom meeting transcript, caption file or AI Companion summary
  into checked action items for DD CPQ. Reads the transcript from a
  pasted block, a local file, or the Google Drive "Meeting recordings"
  folder. Every decision is verified against the actual repo and org
  before it is written down, so a meeting that agreed something already
  done, already impossible, or contradicting a hard rule says so. Emits
  owner-tagged actions, a contradictions list, and draft memory entries.
---

# Meeting → verified action items

A meeting produces claims. This skill turns them into **checked** claims.

The value is not transcription — Zoom already did that. It is that a room
full of people can confidently agree something that is already built, already
ruled out, or flatly contradicted by the code. Writing those down unchecked
turns a wrong belief into a work item, and somebody then spends a day on it.

## 1. Get the transcript

Try these in order, and say which one you used.

| Source         | How                                                                                             |
| -------------- | ----------------------------------------------------------------------------------------------- |
| Pasted in chat | Use it directly. Simplest and always works.                                                     |
| A local file   | `.vtt`, `.txt`, `.srt` or `.md` — read it.                                                      |
| Google Drive   | Search the **Meeting recordings** folder. Zoom cloud recordings land there as dated subfolders. |
| Notion         | `notion-query-meeting-notes`, filtered by date or attendee.                                     |

**If you only find `.mp4` / `.m4a` / `recording.conf`, stop.** That is a Zoom
_local_ recording and it contains no transcript — there is nothing to read,
and no amount of searching will produce one. Say so plainly and point at
Zoom's _Settings → Recording → Cloud recording → Create audio transcript_,
or its automated-captions save option. Do not guess at meeting content from
a filename or a calendar title.

## 2. Pull out only what was actually decided

Ignore chatter, scheduling and thinking-aloud. Keep three kinds of statement:

- **Decisions** — "we will ship X", "we are dropping Y"
- **Actions** — something someone committed to doing
- **Claims about the system** — "the cart already does X", "Y is broken"

Attribute each to whoever said it. Where nobody owned an action, mark the
owner **UNASSIGNED** rather than inventing one — an unowned action is a real
finding about the meeting.

## 3. Check every one against reality — this is the point

Before writing an item down, verify it. Use the repo and the org, not memory:

- `grep`/`Glob` the Apex, LWC and metadata for the thing being discussed
- `sf data query -o cpq-dev` for org state
- `dd_cpq_*` MCP tools for engine behaviour and live numbers
- `CLAUDE.md` for the hard rules

Tag each item with what you found:

| Tag                 | Meaning                                                |
| ------------------- | ------------------------------------------------------ |
| **CONFIRMED**       | The premise holds. Safe to action.                     |
| **ALREADY DONE**    | It exists. Name the file or record proving it.         |
| **CONTRADICTED**    | The code or org says otherwise. Show the evidence.     |
| **BLOCKED BY RULE** | A CLAUDE.md hard rule forbids it. Quote the rule.      |
| **UNVERIFIABLE**    | Cannot be checked from here. Say what would settle it. |

Never silently fix a wrong premise. Report it — the room believed something
untrue, and that is worth more than the action item.

Watch for these, which come up repeatedly in DD CPQ meetings:

- Anything implying a **deploy from `deepdive-cpq`** — frozen and archived.
- Anything assuming **Constitutional CPQ, Agreement Studio, Amendments,
  Migrator, DocRaptor, Deal Scoring or Guided Selling** — all cut, and their
  objects deleted from `cpq-dev` on 2026-09-18.
- Anything assuming a committed quote flows to an **Order** — it does not on
  a stock org; see the correction under CLAUDE.md rule 8.
- A new capability agreed as "just an LWC" — rule 9 requires LWC + REST + MCP.

## 4. Output

Report in this order, and keep it short enough to read in one go:

1. **Decisions** — what was settled, one line each, with who decided.
2. **Actions** — owner, what, and its verification tag.
3. **Contradictions** — the items where the room was wrong. Lead with the
   evidence, not the opinion. If this list is empty, say so explicitly.
4. **Open questions** — what was raised and not resolved.
5. **Draft memory entries** — for decisions that change how the project
   works, propose a memory file following the house format. Do not write
   memory without the user confirming.

Then ask which actions to take now. Do not start work off the back of a
transcript without the user picking — a meeting is evidence of intent, not
an instruction to you.

## 5. Do the work

For each action the user picks, follow the normal rules of this repo:
branch hygiene, `-d force-app` on every deploy, tests before claiming done,
and report the actual result including failures.
