---
name: add-to-backlog
description: >
  Capture a new item into the kanban backlog with minimal friction. Use this skill when the
  user says "add to backlog", "add backlog item", "new backlog", "capture this", "queue this",
  or otherwise wants to record an idea/task without starting work on it immediately.
  Creates a DailyPlanner task and appends a board entry to backlog.md with a trace ID.
---

# Add to Backlog

Quick-capture skill for new ideas, tasks, bugs, and features. Creates a fully-tracked entry in **one round-trip** with the user, then both DailyPlanner and the board reflect it.

## When to Use

| User says | Action |
|---|---|
| "add to backlog: {idea}" | Use the text after the colon as the title; ask for missing details |
| "queue this: {idea}" | Same as above |
| "capture this idea: {idea}" | Same as above |
| "add backlog item" | Ask for title + description |
| "I want to do X someday" | Confirm "Add to backlog?", then proceed |
| "remind me to {do thing}" | Confirm "Track in backlog?", then proceed |

## Instructions

### Step 1: Resolve the board

Reuse the `resolve-board-root` helper from the **start-task** skill ([Board Conventions](../start-task/SKILL.md#board-conventions)):
- All boards resolve to `C:\boards\<board-name>\` — code repos and non-code cwd alike.
- The board name comes from the `.board` pointer file at the repo/cwd root, falling back to the repo directory name, then to a user prompt for fresh non-code cwd.
- If `C:\boards\` itself doesn't exist yet, it is initialized as a git repository ([Central Boards Repository](../start-task/SKILL.md#central-boards-repository)).
- Bootstrap the per-board folder + 5 stage files + `attachments/` + `.locks/` + `README.md` in parallel if missing.
- Create the `.board` pointer file at the repo/cwd root if missing.

**Non-code cwd (e.g., `personal-copilot`):** when the cwd has no git root, ask the user once:
```
ask_user:
  question: "No git repo here. What board should this idea live in?"
  choices: ["personal-copilot (current cwd)", "<cwd basename> (Recommended)", "personal"]
```
Cache the chosen name in `.board` at the cwd root so the prompt never repeats.

### Step 2: Gather the minimum required fields

Ask the user (only for fields not already inferable from their request):

| Field | Required | Default if not given |
|---|---|---|
| **Title** | yes | — must be provided |
| **Brief description** | yes | If user only gave a title, ask: *"One sentence on what this is about?"* |
| **Priority** | no | `P3` (medium) |
| **Type** | no | Inferred from title keywords (bug→Bug, feature/build→Feature, else Task) |
| **Tags** | no | Inferred from cwd context (e.g., repo name) + keyword scan |
| **Due date** | no | none |
| **Reference images / screenshots** | no | none — see Step 2b |
| **DoD sub-sections** (P1/P2 only) | conditional | see Step 2c |

Use `ask_user` with **multiple-choice** for priority and type to keep the capture fast. Only ask one question at a time. Skip questions where the user already provided the answer in their initial message.

### Step 2b: Capture reference images (optional but encouraged for Features/Bugs)

Ask once:
> *"Any screenshots or reference images for context? Paste paths separated by commas, or 'none'."*

For each provided path:
1. Verify the file exists and is an image (`.png` / `.jpg` / `.jpeg` / `.gif` / `.webp` / `.svg`).
2. Ensure the target folder exists: `C:\boards\<board-name>\attachments\<traceId>\` (created lazily on first add).
3. Compute the destination filename:
   - `<seq>-<slug>.<ext>` where `<seq>` is `1`, `2`, … in capture order and `<slug>` is the original filename's stem, slugified (lowercase, non-alphanum → hyphen, collapsed).
   - Example: `C:\Users\Pius\Pictures\dashboard-mockup.png` → `1-dashboard-mockup.png`.
4. Copy (don't move) the file into the destination — the user keeps the original.
5. Append to the entry YAML under `attachments:`:
   ```yaml
   attachments:
     - file: "attachments/<traceId>/1-dashboard-mockup.png"
       caption: "<user-supplied or filename stem>"
       addedAt: "{now}"
   ```
6. Render the image reference in the entry body under the **Images / References** section:
   ```markdown
   ![1-dashboard-mockup.png — Desired layout](attachments/<traceId>/1-dashboard-mockup.png)
   ```

If the user wants to add an image later, the `start-task` / `review-backlog` skills can also accept attachments — the `attachments/` folder is shared and the `record-activity(taskId, kind="attachment-added", ...)` helper handles the update + commit.

### Step 2c: Capture Definition of Done sub-sections (P1/P2 Bugs/Features)

For P1/P2 Bug or Feature items, walk through three quick prompts. P3/P4 items skip this step — `review-backlog` Plan mode will fill them in when the item is promoted.

Ask one at a time:
1. **Acceptance Criteria**: *"List 2–4 user-visible outcomes that prove this works (one per line)."*
2. **Validation Plan**: *"How will we know — post-deploy — that this is working? Think log lines, dashboard metrics, manual smoke steps."*
3. **Observability Plan**: *"What logs / metrics / traces must this feature emit so we can debug it later? Name each one and the debug question it answers."*

Skip the Test Plan at capture time — it's typically filled in during planning when the implementer knows which test projects apply. Leave a placeholder.

If the user defers (`"skip — I'll plan it later"`), still create all four `### Acceptance Criteria / ### Validation Plan / ### Test Plan / ### Observability Plan` headings in the entry body with a `_Not yet elaborated._` placeholder under each. This guarantees `review-backlog` always has the right structure to fill in.

### Step 3: Create in DailyPlanner

Call `DailyPlanner-create_task` with:
- `title`, `description`, `priority`, `type`, `tags`, `dueDate`
- This returns the DP task with the canonical `taskId` (ObjectId) and integer ID.

**DailyPlanner is created first** so the board entry can reference real IDs.

### Step 4: Generate a trace ID

Generate a short UUID-style identifier:
- Format: `tr-` + 6 lowercase hex characters (e.g., `tr-a8f3d2`)
- Must be unique across all stage files — grep all 5 stage files for the candidate before committing. Regenerate on collision.
- The trace ID is **immutable** for the life of the entry; it follows the task across systems (board, DP comments, commit messages, PR descriptions).

### Step 5: Build & append the backlog entry

Construct the entry per the [Entry Format](../start-task/SKILL.md#entry-format) with these defaults for a fresh backlog item:

```yaml
taskId: {integer from DP}
traceId: tr-{6 hex}
title: "..."
priority: P3
type: Task
tags: [...]
workflow: null                              # not yet detected; review-backlog will fill
stage: backlog

parallelizable: true                        # default; review-backlog may set false
mutates:
  repos: ["<board-name>"]                   # current board
  paths: []                                 # populated during review-backlog Plan mode

elaboration:
  level: minimal                            # captured, not yet planned
  estimatedEffort: null
  proposedWorkflow: null
  acceptanceCriteria: {N from Step 2c, else 0}
  validationPlanned: {true if Step 2c populated, else false}
  testPlanned: false                        # always false at capture; filled at planning
  observabilityPlanned: {true if Step 2c populated, else false}
  capturedBy: add-to-backlog

session:
  id: null                                  # no session yet — not started
  name: null
  worktree: null
  branch: null

git:
  branch: null
  baseBranch: main
  commits: []
  pullRequests: []

dailyPlanner:
  taskId: "..."
  outcomeId: null
  status: "New"
  lastSyncedAt: "{now}"

timestamps:
  createdAt: "{now}"
  startedAt: null
  blockedAt: null
  reviewAt: null
  completedAt: null

filesTouched: []
relatedTasks: []
attachments: [...]                          # from Step 2b; [] if none
```

Body sections (always emit all of these, even when empty — `review-backlog` relies on the structure):

```markdown
**Description**

{User-provided description}

**Images / References**

{Markdown image refs from Step 2b; if none: `_(none)_`}

**Definition of Done**

### Acceptance Criteria
{Bulleted checklist from Step 2c, or `_Not yet elaborated. Run `review-backlog` → "plan this item" to flesh out._`}

### Validation Plan
{From Step 2c, or `_Not yet elaborated._`}

### Test Plan
_Not yet elaborated._

### Observability Plan
{From Step 2c, or `_Not yet elaborated._`}

**Notes & Decisions**

_(none yet)_

**Activity Log**

- {timestamp} — Captured via add-to-backlog; traceId={traceId}{; attached N image(s) if any}
```

Append the entry to `backlog.md` (after the file header, separated by `---`). Then call `commit-board-change(<board>, "[{taskId}] add-to-backlog: {short-title}")` so the new entry is persisted to the central boards repo immediately. ([Helper Operations](../start-task/SKILL.md#helper-operations))

### Step 6: Confirm

Output to the user:

```
✅ Added to backlog
   Title:    {title}
   Task ID:  [{integer}]   (DailyPlanner: {ObjectId})
   Trace ID: {traceId}
   Stage:    backlog
   Board:    {path to backlog.md}

Next: run `review backlog` to plan this item, or `start task {integer}` to begin work.
```

## Batch Capture

If the user provides multiple items in one message (e.g., "add these to backlog: 1) X, 2) Y, 3) Z"):

1. Parse the list.
2. **Run Steps 3–5 in parallel** for all items: issue all `DailyPlanner-create_task` calls in one batch (max 10 per turn), then build all entries, then write `backlog.md` in a single edit that appends them all.
3. Confirm with a summary table:
   ```
   ✅ Added 3 items to backlog
      [142] Build Email Template     tr-a8f3d2
      [143] Wire Stripe webhook      tr-b1c4e5
      [144] Audit logger output      tr-c9d7f0
   ```

## Defaults & Inference

When the user is terse, infer rather than ask:

| Heuristic | Inference |
|---|---|
| Title contains "fix", "bug", "broken", "crash" | `type: Bug`, `priority: P2`, `tags: [bugfix]` |
| Title contains "investigate", "research", "explore" | `type: Task`, `tags: [research]` |
| Title contains "document", "docs", "guide" | `type: Task`, `tags: [docs]` |
| Title contains "build", "create", "implement" | `type: Feature`, `tags: [engineering]` |
| Cwd is a code repo | Add the repo name as a tag |
| User says "urgent" or "asap" | `priority: P1` |
| User says "someday" or "nice to have" | `priority: P4` |

Always show the inferred values in the confirmation; the user can correct them with a follow-up like "make it P1" or "add tag frontend".

## Edge Cases

| Situation | Handling |
|---|---|
| **Board folder missing** | Bootstrap silently (Step 1); seed/migration only triggers on first existence |
| **DailyPlanner unreachable** | Append to `backlog.md` with `dailyPlanner.taskId: null` and `lastSyncedAt: null`. Print warning. The next `review-backlog` or `start-task` run will retry sync. |
| **Trace ID collision** | Regenerate up to 5 times; if still colliding, fall back to `tr-{8 hex}` |
| **User adds duplicate title** | Search `backlog.md` for matching title; warn and ask "Add anyway, or update existing?" |
| **Cwd has no clear context** (random dir) | Ask once for a board name (default suggestion: cwd basename); cache in `.board`; tag with `inbox` so `review-backlog` can suggest re-homing later |
| **User pastes image path that doesn't exist** | Skip that one with a warning; continue with the remaining attachments |
| **Attachment folder write fails** (disk full, perms) | Surface error; offer to retry or skip attachments; still create the entry |
| **P1/P2 Bug/Feature with user saying "skip DoD"** | Honor it but emit all four DoD headings with `_Not yet elaborated._` placeholders so structure is preserved |

## Critical Rules

1. **DailyPlanner first, board second** — never write a board entry without a DP task ID (unless DP is unreachable, in which case mark `lastSyncedAt: null`).
2. **One entry per item** — if a duplicate is detected, ask before adding.
3. **Trace ID is immutable** — never regenerate it after the entry is written.
4. **Capture is fast** — minimize questions. The user can always enrich later via `review-backlog`.
5. **Stage is always `backlog`** — this skill never writes to other stage files.
