---
name: get-tasks
description: List tasks from the Daily Planner with full details including status, priority, due dates, subtasks, and AI prompts. Use this skill when the user asks to get tasks, list tasks, show tasks, check tasks, what needs to be done, or view the backlog.
---

# Get Tasks from Daily Planner

Retrieve and display tasks from the Daily Planner with comprehensive details.

## Instructions

Use the `DailyPlanner-get_tasks` tool to fetch tasks. Apply filters based on the user's request:

### Default behavior (no specific filter requested)
Show all non-completed tasks:

1. Call `DailyPlanner-get_tasks` with `status: "all"` (or omit for default)
2. For any task that has `AI Prompt: ✅ Available`, call `DailyPlanner-get_task_prompt` with the task ID to retrieve the full prompt
3. Also call `DailyPlanner-get_task` for tasks marked as today's focus or in-progress to get subtask details

### Available filters
Use these based on user request:
- **Status**: `New`, `InProgress`, `Complete`, or `all`
- **Priority**: `P1`, `P2`, `P3`, `P4`
- **Type**: `Task`, `Feature`, `Product`, `Bug`, `UseCase`, `Scenario`
- **Due date**: `today`, `overdue`, `week`, `none`, `all`
- **Today's focus**: `isToday: true`
- **Tag**: filter by tag name (e.g., "Trading Management System")

### Display format

Present tasks grouped by status, showing:

**For each task:**
- Title (with task ID)
- Status, Priority, Type
- Due date (highlight overdue in bold)
- Tags
- Subtask progress (e.g., "2/5 done")
- Whether it's marked for Today's Focus (📌)
- AI Prompt summary (if available — fetch with `DailyPlanner-get_task_prompt`)

**Grouping order:**
1. 📌 Today's Focus
2. 🔴 Overdue
3. 🔄 In Progress
4. 📋 New / Backlog
5. ✅ Completed (only if explicitly requested)

### Getting suggested focus

If the user asks "what should I work on?" or "what's next?", also call `DailyPlanner-get_suggested_focus` to get AI-ranked task recommendations.

### Examples of user requests mapped to filters

| User says | Filter to use |
|---|---|
| "get tasks" | `status: "all"` (exclude Complete) |
| "show my tasks for today" | `isToday: true` |
| "what's overdue?" | `dueDate: "overdue"` |
| "show P1 tasks" | `priority: "P1"` |
| "what bugs do I have?" | `type: "Bug"` |
| "show completed tasks" | `status: "Complete"` |
| "tasks due this week" | `dueDate: "week"` |
| "trading tasks" | `tag: "Trading Management System"` |

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
  description = "Surfaced by: get-tasks · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "get-tasks"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.