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

Reuse the `resolve-board-root` helper from **start-task** ([Board Conventions](../start-task/SKILL.md#board-conventions)). All boards live at `C:\boards\<board-name>\`. Bootstrap if missing (creates the central boards repo, the per-board folder, and the `.board` pointer file at the repo/cwd root). If the board is empty, inform the user and suggest `add-to-backlog`.

### Step 1: Detect mode from user request

| Signal | Mode |
|---|---|
| No specific id mentioned | **List** |
| Specific id + "show", "open", "view" | **Inspect** |
| Specific id + "plan", "flesh out", "elaborate", "groom", "detail" | **Plan** |
| Specific id + "drop", "archive", "remove" | **Drop** |
| Specific id + "start", "begin", "promote" | Hand off to `start-task` |

If unclear, default to **List** with the option to drill in.

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

### Step P4: Update the entry in place

Apply all answers to the YAML and prose sections:
- Update `elaboration.level` (`minimal` → `partial` or `full` depending on completeness)
- Update `elaboration.estimatedEffort`, `elaboration.proposedWorkflow`, `elaboration.acceptanceCriteria` (count)
- Replace **Description** if expanded
- Append acceptance criteria to **Definition of Done**
- Append a `Notes & Decisions` line for any decision captured (e.g., chosen workflow, related-task links)
- Append to **Activity Log**: `{timestamp} — Planned via review-backlog; elaboration: {level}`
- Update `relatedTasks: [...]` if links were established
- Push changes to DailyPlanner via `sync-with-daily-planner(taskId, mode=push)` ([sync rules](../start-task/SKILL.md#bi-directional-sync-with-dailyplanner))
- Persist to the central boards repo via `commit-board-change(<board>, "[{taskId}] review-backlog: elaborate to {level}")` ([Helper Operations](../start-task/SKILL.md#helper-operations))

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
