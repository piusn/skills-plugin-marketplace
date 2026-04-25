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
