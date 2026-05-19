---
name: quick-task
description: >
  Quickly add a task to the Daily Planner with automatic outcome assignment.
  Use this skill when the user says 'quick task', 'add task', 'new task',
  'create task', 'quick add', 'add to planner', or describes a task they want
  to add without going through the full start-task workflow. Parses the input,
  matches to the best existing outcome, and creates the task — all in one step.
---

# Quick Task — Fast Task Creation with Auto Outcome Assignment

Add a task to the Daily Planner in seconds. The skill figures out which outcome to place it under (or creates a new one) so every task is properly organized.

## When to Use

- You want to quickly capture a task without the full `start-task` ceremony
- You have a task idea and want it in the planner immediately
- You want the system to figure out where it belongs (which outcome)

## When NOT to Use

- You want to **start working** on a task right now → use `start-task` instead
- You need to create a complex task with subtasks, checklists, or AI prompts → create manually in the Daily Planner UI

## Instructions

### Step 1: Parse the User's Input

Extract from the user's message:

| Field | Required | Default | Examples |
|-------|----------|---------|---------|
| **Title** | ✅ Yes | — | "Fix the login timeout bug", "Write API docs for search endpoint" |
| **Description** | No | Empty | Additional context the user provides |
| **Priority** | No | P4 | P1, P2, P3, P4 — infer from urgency words if not stated |
| **Type** | No | Task | Task, Feature, Bug, Product, UseCase, Scenario |
| **Tags** | No | Empty | Comma-separated, infer from context |
| **Due date** | No | None | "by Friday", "next week", "2026-05-15" |
| **isToday** | No | false | Only true if user explicitly says "for today" or "today's focus" |

**Priority inference rules** (when not explicitly stated):
- Words like "urgent", "critical", "ASAP", "blocking", "production" → P1
- Words like "important", "high priority", "this week" → P2
- Words like "should", "would be nice", "when possible" → P3
- Default when no signal → P4

**Type inference rules** (when not explicitly stated):
- Words like "fix", "bug", "broken", "regression", "issue" → Bug
- Words like "build", "implement", "create", "add feature" → Feature
- Words like "product", "launch", "release" → Product
- Default → Task

### Step 2: Fetch Active Outcomes

Call `DailyPlanner-get_outcomes` with `status: "Active"` and `includeArchived: false` to retrieve only active outcomes.

This returns each outcome's:
- ID, title, description
- Tags
- Task count and progress
- Whether it's the default outcome (`isDefault`)
- Goal linkage

### Step 3: Match Task to Best Outcome

Evaluate each active outcome against the task to find the best fit. Use this scoring approach:

#### Matching Criteria (in priority order)

1. **Tag overlap** (strongest signal)
   - If the task has tags that match an outcome's tags → strong match
   - Example: Task tagged "API" matches outcome tagged "API Platform"

2. **Semantic title/description match** (primary reasoning)
   - Does the task logically belong under this outcome's stated purpose?
   - Example: "Fix login timeout" → outcome "Improve Authentication Reliability"
   - Example: "Write API docs" → outcome "API Documentation"

3. **Outcome specificity** (prefer specific over broad)
   - An outcome titled "Improve Search Performance" is better for a search-related task than "General Engineering"
   - Avoid putting tasks in overly broad outcomes when a specific one fits

4. **Outcome activity** (prefer active outcomes)
   - Outcomes with recent tasks and ongoing progress are better candidates than stale/empty ones
   - An outcome with 0 tasks might be new and waiting for tasks, or it might be abandoned

5. **Default outcome** (last resort)
   - If an outcome has `isDefault: true`, use it only when no other outcome fits AND the task is generic/administrative
   - For tasks that clearly represent a new workstream or initiative, prefer creating a new outcome over using the default

#### Match Confidence Levels

| Confidence | Action |
|------------|--------|
| **High** — clear semantic fit, tag overlap, or obvious match | Create immediately, report result |
| **Medium** — reasonable fit but not obvious | Create immediately, but mention the reasoning |
| **Low** — weak fit, nothing clearly matches | Proceed to Step 4 (create new outcome) |
| **None** — no outcomes exist or all are clearly unrelated | Proceed to Step 4 (create new outcome) |

### Step 4: Create New Outcome (When No Match Found)

When no existing outcome is a good fit, propose a new one.

#### Generate a Good Outcome Title

The outcome should be:
- **Goal-oriented** — describes a desired result, not just a category
- **Broader than the task** — an outcome should accommodate multiple related tasks
- **Action-oriented** — starts with a verb or implies progress toward a goal

**Good outcome titles:**
- "Improve API Reliability" (not "API Tasks")
- "Launch User Dashboard MVP" (not "Dashboard Work")
- "Strengthen Security Posture" (not "Security Stuff")
- "Modernize Authentication System" (not "Auth")
- "Complete Q2 Documentation" (not "Docs")

**Bad outcome titles:**
- "Miscellaneous" — too vague
- "Fix login bug" — too specific (that's a task, not an outcome)
- "Work" — meaningless

#### Confirm with User Before Creating

Since creating a new outcome is a structural change, confirm first:

```
ask_user:
  question: "No existing outcome fits this task well. I'd like to create a new outcome:"
  choices:
    - "Create: '{proposed outcome title}' (Recommended)"
    - "Let me specify a different outcome name"
    - "Skip outcome — just create the task without one"
```

If the user provides a custom name, use that instead.

#### Create the Outcome

Call `DailyPlanner-create_outcome` with:
- `title`: The confirmed outcome title
- `description`: A brief description of what this outcome covers (1-2 sentences)
- `status`: `"Active"`
- `priority`: Match the task's priority, or default to P3
- `tags`: Relevant tags inferred from the task context

**Capture the returned outcome ID** — you need it when linking the task in Step 6.

### Step 5: Create the Task

Call `DailyPlanner-create_task` with:
- `title`: The parsed task title
- `description`: The parsed description (if any)
- `priority`: The parsed or inferred priority
- `type`: The parsed or inferred type
- `dueDate`: The parsed due date (if any)
- `tags`: The parsed or inferred tags
- `isToday`: false (default) unless user explicitly requested

**Capture the returned task ID** — you need it for the next step.

### Step 6: Link Task to Outcome

Call `DailyPlanner-update_task` with:
- `taskId`: The ID returned from Step 5
- `outcomeId`: The matched or newly created outcome ID

**If linking fails:** Report that the task was created but could not be linked. Include both the task ID and the intended outcome so the user can retry manually.

### Step 7: Confirm to User

Present a clean summary:

```markdown
✅ Task added to Daily Planner!

📋 **{task title}**
   Priority: {priority} | Type: {type}
   {Due: yyyy-MM-dd (if set)}
   {Tags: tag1, tag2 (if any)}

📎 Outcome: **{outcome title}** {(new) if just created}
   {Progress: X/Y tasks completed}

💡 To start working on it: `start task {task number}`
   To mark for today: `update task {task id} isToday true`
```

## Handling Multiple Tasks

If the user provides multiple tasks in one message (e.g., "add these tasks: X, Y, Z"), process each one efficiently:

1. Fetch outcomes once (Step 2)
2. For tasks with clear outcome matches, create and link immediately
3. For tasks needing new outcomes, batch the proposals — present all proposed outcomes in one confirmation prompt rather than asking one-by-one
4. Present a consolidated summary at the end

## Edge Cases

| Scenario | Handling |
|----------|---------|
| User says just "quick task" with no details | Ask: "What task do you want to add?" |
| DailyPlanner tools not available | Inform user: "Daily Planner MCP is not available. Please ensure it's running." |
| Task creation fails | Report the error and suggest retrying |
| Outcome creation fails | Create the task without an outcome, inform user |
| All outcomes are Completed/Archived | Treat as no match — propose a new outcome |
| User says "skip outcome" | Create task without linking to any outcome |
| Task already exists (similar title) | Don't check for duplicates — the user may intentionally want similar tasks |

## Tools Used

| Tool | Purpose |
|------|---------|
| `DailyPlanner-get_outcomes` | Fetch active outcomes for matching |
| `DailyPlanner-create_outcome` | Create new outcome when no match found |
| `DailyPlanner-create_task` | Create the task |
| `DailyPlanner-update_task` | Link task to outcome via `outcomeId` |
| `ask_user` | Confirm new outcome creation |

## Examples

### Example 1: Clear match to existing outcome

**User:** "quick task: fix the retry logic in the payment gateway"

**Outcomes fetched:**
- "Improve Payment System Reliability" (Active, tags: payments, reliability)
- "Launch User Dashboard MVP" (Active, tags: dashboard, frontend)

**Action:** High confidence match → "Improve Payment System Reliability"
**Result:** Task created and linked immediately, no confirmation needed.

### Example 2: No matching outcome

**User:** "add task: set up Grafana dashboards for the new API"

**Outcomes fetched:**
- "Complete Q2 Documentation" (Active)
- "Improve Authentication System" (Active)

**Action:** No match → propose new outcome "Establish API Observability"
**Confirmation:** Ask user to confirm the new outcome name.
**Result:** New outcome created, task created and linked.

### Example 3: Minimal input

**User:** "quick task: update README"

**Action:** Match to a documentation-related outcome if one exists, otherwise propose "Improve Project Documentation".
**Priority:** P4 (no urgency signal)
**Type:** Task (default)

### Example 4: Rich input

**User:** "add task: implement rate limiting for the search API — P1, due Friday, tag: security, api"

**Parsed:**
- Title: "Implement rate limiting for the search API"
- Priority: P1
- Due: next Friday's date
- Tags: security, api
- Type: Feature (inferred from "implement")

**Action:** Match to an API or security outcome based on tags and context.

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
  description = "Surfaced by: quick-task · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "quick-task"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.