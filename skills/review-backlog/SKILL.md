---
name: review-backlog
description: >
  Review and refine items in the kanban backlog. Use this skill when the user says
  "review backlog", "show backlog", "what's in the backlog", "plan backlog item",
  "flesh out backlog", "groom backlog", "pick a backlog item", or wants to inspect,
  enrich, or promote backlog items into work-ready tasks.
---

# Review Backlog

Three-mode skill for working with the backlog: **list** (overview), **inspect** (details), and **plan** (flesh out a thin idea into a work-ready task).

> **🚀 MCP-ONLY MODE (issue #37, effective 2026-05-19):**
> Daily Planner is the **sole** source of truth. The on-disk file board
> under `C:\boards\<group>\*.md` is **retired** — do not read or write it.
> The "resolve the board" / `.board` pointer file dance below is **deprecated**.
>
> **New canonical flow:**
>
> - **List**: `DailyPlanner-get_tasks` filtered by `stage='backlog'` and optionally `group='<board-name>'`. The default `group` is the cwd basename; the user can override.
> - **Inspect**: `DailyPlanner-get_task(id)` returns the full task with all subdocuments (`BoardMetadata`, `DefinitionOfDone`, `DevMetadata`, `NotesAndDecisions`).
> - **Plan**:
>   - `DailyPlanner-update_task_board_metadata(taskId, level, estimatedEffort, proposedWorkflow, parallelizable, mutates.repos, mutates.paths)` — elaboration update.
>   - `DailyPlanner-update_task_definition_of_done(taskId, ...)` — DoD subdocument.
>   - `DailyPlanner-append_task_note(taskId, content, kind='Decision')` — decision logging.
>   - `DailyPlanner-link_related_tasks(taskId, otherTaskId)` — related items.
> - **Drop**: `DailyPlanner-move_task_stage(taskId, 'completed')` + `DailyPlanner-update_task(id, status='Cancelled')`. The state-rule contract auto-marks `Completed=true`.
>
> **📌 Daily Planner repo special-case**: When the cwd is `daily-planner` or any Sokokapu-Limited microservice repo, also list open tasks with `tags=mcp-gap` and surface them at the top of the list so we keep closing capability gaps as we work.

---

## 🩺 Health check on entry (issue #43, effective 2026-05-19)

Before doing any backlog work, run a quick state check and surface issues:

1. `DailyPlanner-get_outcomes()` — count outcomes where `status='Active'` and `kind='Deliverable'`.
   - If count **> 5**: warn the user. *"You have N active outcomes; the soft cap is 5. Want to archive some before adding more work?"* Suggest `batch_archive_outcomes`.
2. Scan the backlog for **recurring patterns** (`Daily X`, `Weekly X`, `Morning X`, `Evening X`). Surface these as candidates for habit/routine conversion:
   - *"These N tasks look recurring — consider `convert_task_to_habit` or `convert_task_to_routine_item` to move them out of the backlog."*
3. List tasks with `status='Inbox'` separately at the top — the user should triage these before planning further.

This isn't gatekeeping — it's a 30-second sanity check that keeps the backlog small and signals when the user's intent might fit a better entity (habit/routine).

---

## Legacy disk flow (retired — reference only)

The sections below describe the pre-issue-#37 flow that combined MCP + disk
snapshots and the `resolve-board-root` helper. They are preserved for context
only — **do not follow them**. The MCP-only flow above supersedes everything below.



## When to Use

| User says | Mode |
|---|---|
| "review backlog" / "show backlog" / "what's in the backlog" | List |
| "what's in {tag/priority} backlog" | List (filtered) |
| "show me backlog item {id}" / "open [142]" | Inspect |
| "plan backlog item {id}" / "flesh out [142]" / "groom [142]" | Plan |
| "elaborate on the backlog" / "groom the backlog" | List → Plan iterating |
| "promote {id} to in progress" | Plan (lite) → handoff to `start-task` |
| "drop {id}" / "archive {id}" | Drop |

## Instructions

### Step 0: Resolve the board

Reuse the `resolve-board-root` helper from **start-task** ([Board Conventions](../start-task/SKILL.md#board-conventions), [Board Name Derivation](../start-task/SKILL.md#board-name-derivation)).

⛔ **The helper never guesses which board to use.** When the cwd / repo has no `.board` pointer file, it prompts the user explicitly with the list of existing boards under `C:\boards\` plus options to create a new board or to cancel. Auto-deriving the board from the repo directory name is **forbidden**.

If `resolve-board-root()` returns `null` (user chose "Continue without a board"), exit this skill immediately:

```
⛔ No board configured for this repository.
   Re-run `review backlog` once you've either created a board or pointed the
   repo at an existing one (e.g. write the path to `<repo-root>/.board`).
```

Otherwise: all boards resolve to `C:\boards\<board-name>\`. Bootstrap if missing (creates the central boards repo, the per-board folder, and the `.board` pointer file at the repo/cwd root). If the board is empty, inform the user and suggest `add-to-backlog`.

### Step 1: Detect mode from user request

| Signal | Mode |
|---|---|
| No specific id mentioned | **List** |
| Specific id + "show", "open", "view" | **Inspect** |
| Specific id + "plan", "flesh out", "elaborate", "groom", "detail" | **Plan** |
| Specific id + "drop", "archive", "remove" | **Drop** |
| Specific id + "start", "begin", "promote" | Hand off to `start-task` |

If unclear, default to **List** with the option to drill in.

### Step 1.5: Detect Daily Planner context and pre-fetch MCP gaps

Before running the chosen mode, detect whether the user is working inside the
Daily Planner ecosystem. If **any** of the following is true, set
`dpContext = true`:

- `cwd` is, or is under, `C:\repositories\Sokokapu-Limited\daily-planner\`
- `cwd` is, or is under, any sibling Sokokapu microservice repo under
  `C:\repositories\Sokokapu-Limited\` (habit-management-system,
  productivity-management-system, finance-management-system,
  exercise-managment-system, goal-management-system,
  journalling-management-system, learning-management-system,
  wellness-management-system, planning-management-system,
  reporting-management-system)
- The resolved board name from Step 0 matches `daily-planner` or any
  microservice repo name
- `git remote -v` of the cwd points at `github.com/Sokokapu-Limited/*`

When `dpContext = true`, fetch open MCP gap items in parallel with whatever
the chosen mode is about to do:

```
DailyPlanner-list_tasks(tags="mcp-gap", status="!Completed")
```

Keep the result in memory as `mcpGaps` for use by the rendering steps below.
If the call fails, log a single warning line and continue — do not block the
chosen mode.

> Why this exists: every skill that touches Daily Planner is instructed to
> capture friction as `mcp-gap`-tagged backlog items (see
> [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md)). Surfacing
> them here closes the loop so the MCP surface keeps improving.

---

## Mode: List

### Step L1: Read all backlog entries

`grep` `backlog.md` for `## [` and parse each entry's YAML block. **Issue one read; parse all entries from the result.**

### Step L2: Apply optional filters

If the user specified filters (e.g., "P1 backlog", "frontend backlog", "stale backlog"):

| Filter | Match |
|---|---|
| Priority (P1/P2/P3/P4) | `priority` field |
| Tag (frontend, bug, etc.) | `tags` contains |
| Type (Bug, Feature, Task) | `type` field |
| Stale (default: > 30 days) | `timestamps.createdAt` older than threshold |
| Unelaborated | `elaboration.level == "minimal"` |
| Workflow (engineering, quickfix, etc.) | `elaboration.proposedWorkflow` field |

### Step L3: Render the table

**If `dpContext = true` and `mcpGaps` is non-empty**, render the gaps section
first (above the normal backlog table) so it's the first thing the developer
sees:

```
🔧 MCP/API Gaps ({K} open) — surfaced because cwd is in the Daily Planner ecosystem

  ID    Title                                          Priority  Skill            Age
  ────  ─────────────────────────────────────────────  ────────  ───────────────  ─────
  [218] [MCP gap] add bulk_update_tags                 P2        review-backlog    2d
  [221] [MCP gap] task list missing dueDate field      P3        plan-day          1d
  [225] [MCP gap] clarify error on duplicate traceId   P3        add-to-backlog    4h

  Try: "plan [218]" to flesh out a fix, or "start task 218" to begin work.
```

Sort gaps by priority desc, then age desc. Group secondary tags (excluding
`mcp-gap` and `daily-planner`) reveal which skill surfaced each gap.

Then render the normal backlog table below it:

```
📋 Backlog ({N} items, {M} unelaborated)

  ID    Title                              Priority  Type      Age    Elab.  Tags
  ────  ─────────────────────────────────  ────────  ────────  ─────  ─────  ─────────────
  [142] Build Email Template               P2        Feature    3d    full   engineering, frontend
  [143] Wire Stripe webhook                P2        Feature    3d    min    engineering, payments
  [144] Audit logger output                P3        Task       3d    min    docs
  [145] Fix login timeout on mobile        P1        Bug        1d    min    bugfix, mobile

  Sorted by: priority desc, age desc.
  Try: "plan [143]", "show [142]", "drop [144]", "start task 145"
```

- **Age** is human-relative (`1d`, `2w`, `3mo`).
- **Elab.** column shows `min`, `part`, or `full` from `elaboration.level`.
- Highlight **unelaborated P1/P2 items** with a ⚠️ — these are most likely to need attention.

### Step L4: Suggest next action

If unelaborated high-priority items exist, prompt:
> *"3 P1/P2 items are unelaborated. Want to plan them now? (yes / pick one / skip)"*

If `dpContext = true` and any `mcp-gap` items exist with elaboration `minimal`,
also prompt:
> *"{K} MCP/API gaps are unelaborated. Plan one now to improve the Daily Planner surface? (yes / pick one / skip)"*

---

## Mode: Inspect

### Step I1: Locate and parse

Read the entry from `backlog.md` by integer ID or trace ID. Parse YAML + prose sections.

### Step I2: Render full details

```
[142] Build Email Template
─────────────────────────────────────────────────────
Trace ID:    tr-a8f3d2
Priority:    P2          Type: Feature
Tags:        engineering, frontend
Created:     2026-05-12 (3 days ago)  by add-to-backlog
Elaboration: full        Effort: medium  Workflow: engineering-task
Parallelizable: yes      Mutates: trade-management-system: trading-backend/src/Email/**, trading-frontend/src/email-templates/**
DailyPlanner: 65a1f3... (status: New)

Description
  {full description}

Images / References (2)
  • attachments/tr-a8f3d2/1-dashboard-mockup.png — Desired layout from designer
  • attachments/tr-a8f3d2/2-error-state.png      — How the error path should look

Definition of Done
  Acceptance Criteria (3, 0 checked)
    - [ ] Email template loader handles missing templates without crashing
    - [ ] Loader supports nested partials
    - [ ] Output passes Litmus rendering checks
  Validation Plan
    • Manual smoke: render "welcome" template — expect 200 + body
    • Log: INFO Email.TemplateLoader: loaded template=… bytes=…
    • Metric: email_template_render_total{status="success"} increments
  Test Plan
    • Unit: tests/TradingManagement.Application.Tests/Email/TemplateLoaderTests.cs
  Observability Plan
    • Logs: INFO loaded, WARN template_missing, ERROR template_invalid
    • Metrics: email_template_render_total (counter), _duration_ms (histogram)
    • Traces: Email.TemplateLoader.Load span with {template, source}

Notes & Decisions
  - 2026-05-13 — Decided on Handlebars over Liquid

Activity Log
  - 2026-05-12 — Captured via add-to-backlog
  - 2026-05-12 — Attached 2 images
  - 2026-05-13 — Note added: Handlebars decision

Actions: plan / drop / start / edit
```

### Step I3: Wait for user action

Ask: *"What next?"* with choices: `plan`, `start`, `drop`, `back to list`.

---

## Mode: Plan (Flesh Out)

This is the most substantive mode. It transforms a `minimal` entry into `partial` or `full` elaboration through structured Q&A.

### Step P1: Show the current state

Render a brief inspect view (Step I2) so the user sees what's already there.

### Step P2: Identify gaps

Compare current entry against the **planning checklist**:

| Field | Required for `partial` | Required for `full` |
|---|---|---|
| Description (≥ 1 paragraph) | ✅ | ✅ |
| Priority | ✅ | ✅ |
| Tags | ✅ | ✅ |
| `mutates.paths` populated (or explicitly empty for non-code tasks) | ❌ | ✅ |
| `parallelizable` decided | ❌ | ✅ |
| `elaboration.proposedWorkflow` | ✅ | ✅ |
| `elaboration.estimatedEffort` | ✅ | ✅ |
| Acceptance Criteria (≥ 2 items) | ❌ | ✅ |
| Validation Plan populated (logs + metrics + manual checks) | ❌ | ✅ |
| Test Plan populated (test project paths + test types) | ❌ | ✅ |
| Observability Plan populated (logs + metrics + traces + correlation IDs) | ❌ | ✅ |
| Related tasks identified | ❌ | ✅ |
| Risks/unknowns noted | ❌ | ✅ |
| Suggested first commit/sub-task | ❌ | ✅ |

### Step P3: Ask probing questions

Walk through the missing fields **one question at a time** using `ask_user`. Use multiple-choice where possible.

Suggested probing questions:

1. **Description depth**: *"The description is one line. What's the underlying motivation — what problem does this solve?"*
2. **Workflow detection**: *"What kind of work is this?"* → use the [workflow detection table](../start-task/SKILL.md#5-detect-task-workflow-type) from start-task.
3. **Effort estimate**: *"Rough effort?"* → choices: `small (< 1 day)`, `medium (1–3 days)`, `large (> 3 days)`, `unknown — needs research`.
4. **Mutates paths**: *"Which files/folders does this task touch? List globs (e.g., `trading-backend/src/Email/**`)."* — required for parallel-safety detection.
5. **Parallelizable?**: *"Can this run in parallel with other tasks touching different paths in the same repo?"* → choices: `yes (recommended)`, `no — serial only`, `unsure`.
6. **Acceptance criteria**: *"What does 'done' look like for the user? List 2–4 testable, user-visible conditions."*
7. **Validation plan**: *"How will we verify post-deploy this is working? List: manual smoke steps, log lines (with severity + message), metric checks, failure-mode validation."*
8. **Test plan**: *"Which test project(s) get new tests, and which test types (unit/integration/E2E/regression)?"*
9. **Observability plan**:
   - *"Logs: which log lines must this feature emit? List as `<level> <component>: <message-template>` with the debug question each answers."*
   - *"Metrics: which metric names + types + labels? Tie each to the debug question it answers."*
   - *"Traces: any span names + key attributes for distributed flows?"*
   - *"Correlation IDs: which IDs (TaskId, TraceId, PositionId, …) must every log/metric/span carry?"*
10. **Related items**: search the board for similar tags/keywords and ask *"This looks related to [N] — link them?"*
11. **Risks/unknowns**: *"Anything risky, ambiguous, or that needs spike investigation first?"*
12. **First step**: *"What's the smallest first step or commit to make progress?"*

**Skip questions** when the answer is already populated and accurate. Templates for filling out the Observability Plan live in [Observability Templates](../start-task/SKILL.md#observability-templates).

### Step P4: Update the entry — Daily Planner first, then snapshot to disk

**P4a. Push the elaboration to Daily Planner (REQUIRED).** Use the new MCP tools:

1. `update_task_board_metadata(taskId, level={minimal|partial|full}, estimatedEffort={small|medium|large|unknown}, proposedWorkflow=<answer>, parallelizable=<bool>, capturedBy="review-backlog", repos=<comma-list>, paths=<comma-list>)` — single call sets all the planning fields.
2. For each acceptance criterion captured in Step P3.6, call `POST /api/tasks/{taskId}/acceptance-criteria` (or use `update_task_definition_of_done` to set the full DoD subdocument).
3. For each `Notes & Decisions` line captured (e.g. workflow choice), call `append_task_note(taskId, content, kind='Decision')`.
4. For related-task links established in P3.10, call `link_related_tasks(taskId, otherTaskId)` for each.
5. Update DP task description if the description was expanded (existing `DailyPlanner-update_task`).

**P4b. Snapshot to disk (OPTIONAL).** Apply all answers to the YAML and prose sections:
- Update `elaboration.level` (`minimal` → `partial` or `full` depending on completeness)
- Update `elaboration.estimatedEffort`, `elaboration.proposedWorkflow`, `elaboration.acceptanceCriteria` (count)
- Replace **Description** if expanded
- Append acceptance criteria to **Definition of Done**
- Append a `Notes & Decisions` line for any decision captured (e.g., chosen workflow, related-task links)
- Append to **Activity Log**: `{timestamp} — Planned via review-backlog; elaboration: {level}`
- Update `relatedTasks: [...]` if links were established
- Push changes to DailyPlanner via `sync-with-daily-planner(taskId, mode=push)` ([sync rules](../start-task/SKILL.md#bi-directional-sync-with-dailyplanner))
- Persist to the central boards repo via `commit-board-change(<board>, "[{taskId}] review-backlog: elaborate to {level}")` ([Helper Operations](../start-task/SKILL.md#helper-operations))

**All file mutations in this step run under the [Backup & Recovery](../start-task/SKILL.md#backup--recovery) flow**: call `backup-board(<board>, "review-backlog-plan-<taskId>")` before the first edit and `clear-backup(backupPath)` only after `commit-board-change()` returns successfully.

> The disk write is now a snapshot of state that already exists in Daily Planner (P4a). If disk fails, log a warning and continue — the snapshot can be regenerated via `export_board_to_disk`.

### Step P5: Offer next action

```
✅ [142] now elaborated to: full
   Effort: medium   Workflow: engineering-task   Acceptance criteria: 4

Next: start task 142 / plan another / back to list
```

---

## Mode: Drop

### Step D1: Confirm

```
About to drop [142] Build Email Template.
This moves it to completed.md with status=dropped (kept for history; never deleted).

Confirm? (yes / no)
```

### Step D2: Move

`move-task(taskId, "backlog", "completed", { stage: "completed", notes: "Dropped: {reason}", timestamps.completedAt: now, dailyPlanner.status: "Cancelled" })` and sync.

Use the existing `move-task` helper ([Helper Operations](../start-task/SKILL.md#helper-operations)).

---

## Batch Operations

### Bulk plan
*"Plan all P1 backlog items"* — iterate through P1 unelaborated entries, running Plan mode for each. Confirm before moving to next.

### Bulk reprioritize
*"Make all bug-tagged backlog items P2"* — read all entries, filter, apply update, write `backlog.md` in a single edit, push DP updates in batches of 10.

### Bulk drop stale
*"Drop backlog items older than 90 days"* — list candidates first, confirm, then move in batch.

## Parallelization

| Operation | Parallel units |
|---|---|
| List rendering | Single `view` of `backlog.md` (one entry per item; no parallelism needed) |
| Bulk plan | **Sequential per item** — planning is interactive Q&A |
| Bulk update (priority, tags, etc.) | One `edit` per file change; DP updates batched 10/turn |
| Cross-board search (related tasks) | Grep all 5 stage files in **one tool turn** (5 parallel greps) |
| Bulk drop | Multiple `move-task` calls in parallel — each writes to 2 files (backlog.md, completed.md); coordinate so backlog.md is rewritten **once** with all removals |

## Edge Cases

| Situation | Handling |
|---|---|
| **Backlog empty** | Print: *"Backlog is empty. Use `add to backlog` to capture ideas."* |
| **Item not found by ID** | Search across all stage files; if found elsewhere, redirect: *"[142] is in `inprogress.md`, not the backlog. Use `start-task` instead."* |
| **DailyPlanner sync fails during plan** | Save board changes; mark `lastSyncedAt: null`; warn user and continue. Sync will retry next session. |
| **User invokes `start task X` from review** | Hand off to **start-task** skill in the same session; do not duplicate logic. |
| **Trace ID lookup** (user pastes `tr-a8f3d2`) | Grep all stage files for `traceId: tr-a8f3d2` to locate the item, then proceed |
| **Conflicting elaboration** (DP description differs from board) | Apply [Bi-directional Sync conflict rules](../start-task/SKILL.md#conflict-resolution) — most recent wins; surface to user. |

## Critical Rules

1. **Read-modify-write atomicity** — when editing an entry, read the full block, mutate it in memory, write it back. Never partial-edit YAML in place (risk of breaking the YAML).
2. **Plan mode is interactive** — always ask one question at a time; never assume answers.
3. **Drop ≠ delete** — dropped items move to `completed.md` with `dailyPlanner.status: Cancelled`. The history is sacred.
4. **Sync after every change** — every edit triggers a `push` sync to DailyPlanner.
5. **Elaboration level reflects reality** — only mark `full` when all `full` fields are populated. It's better to have honest `partial` than aspirational `full`.
6. **Defer to `start-task` for promotion** — this skill plans; it does not start work.

---

## 🔧 MCP/API Gap Capture

This skill interacts with Daily Planner. While using it, **continuously watch
for friction** with the MCP tools or backend APIs — missing tools, missing
fields, awkward multi-call flows, bad defaults, unclear errors, doc gaps —
and capture each one as a backlog item **inline, without blocking the user's
request**:

```
DailyPlanner-create_task(
  title       = "[MCP gap] <short imperative>",
  description = "Surfaced by: review-backlog · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "review-backlog"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.