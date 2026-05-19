---
description: "Perform daily journaling by compiling the day's activities into a rich journal entry. Use this skill when the user says 'journal', 'daily journal', 'describe my day', 'log my day', 'write journal', or 'journaling'. Pulls work, exercise, meetings, finances, and accepts freeform reflections to create a comprehensive day summary."
---

# Daily Journal Skill

## Context
Daily journaling captures the full picture of the day — not just tasks completed, but also exercise, meetings, financial activity, and personal reflections. This skill enriches the basic `describe_my_day` API with data from all Daily Planner modules.

## When to Use
- At end of day (invoked by `close-day` skill)
- When the user explicitly asks to journal
- When reviewing the day's activities

## Workflow

### Step 1: Gather Day's Data
Pull data from all Daily Planner modules in parallel:

1. **Completed Tasks:**
   ```
   DailyPlanner-get_tasks(status: "Completed", dueDate: "today")
   ```

2. **Today's Meetings:**
   ```
   DailyPlanner-get_todays_meetings()
   ```

3. **Exercise & Workouts:**
   ```
   DailyPlanner-get_exercises(status: "Completed")
   ```
   Filter to today's date from results.

4. **Diet/Nutrition:**
   ```
   DailyPlanner-get_diet_entries(date: "today's date in yyyy-MM-dd")
   ```

5. **Water Intake:**
   ```
   DailyPlanner-get_water_intake(date: "today's date in yyyy-MM-dd")
   ```

6. **Financial Activity:**
   ```
   DailyPlanner-get_expenses(from: "today", to: "today")
   ```

7. **Learning Progress:**
   ```
   DailyPlanner-get_learning_focus()
   ```

### Step 2: Ask for Personal Reflection
Use `ask_user` to prompt for additional input:
- "Any reflections, highlights, or things you want to add to today's journal?"
- This captures context that tools can't — feelings, insights, personal wins

### Step 3: Compose Journal Entry
Structure the journal content with sections:

```markdown
## 📋 Work
- [Completed tasks summary]
- [Key accomplishments]

## 📅 Meetings
- [Meeting summaries with key outcomes]

## 💪 Health & Wellness
- Exercise: [workout details]
- Nutrition: [meals and calories]
- Water: [intake in ml / daily goal]

## 💰 Finances
- [Any expenses or income logged today]

## 📚 Learning
- [Any learning progress]

## 💭 Reflections
- [User's freeform input]
```

### Step 4: Create Journal Entry
```
DailyPlanner-create_journal_entry(
  title: "Day Summary — [date]",
  content: [composed content from above],
  mood: [inferred or asked],
  tags: "daily-journal,work,health"
  date: "yyyy-MM-dd"
)
```

### Step 5: Present Summary
Show the user a clean summary of what was captured, with any gaps flagged:
- "⚠️ No exercise logged today"
- "⚠️ No water intake tracked"
- "✅ 5 tasks completed, 3 meetings attended"

## Tools & APIs Used
- `DailyPlanner-get_tasks` — Completed tasks
- `DailyPlanner-get_todays_meetings` — Meetings
- `DailyPlanner-get_exercises` — Workouts
- `DailyPlanner-get_diet_entries` — Nutrition
- `DailyPlanner-get_water_intake` — Hydration
- `DailyPlanner-get_expenses` — Financial activity
- `DailyPlanner-get_learning_focus` — Learning
- `DailyPlanner-create_journal_entry` — Save journal
- `ask_user` — Get personal reflections

## Output Format
Journal entry created in Daily Planner with structured sections. Summary displayed with gap indicators.

## Mood Detection
If the user doesn't specify a mood, infer from the day's data:
- Many tasks completed + exercise → "productive"
- Fewer completions + reflective notes → "reflective"
- Blocked items + stress indicators → "challenging"
- Always confirm with user before saving

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
  description = "Surfaced by: daily-journal · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "daily-journal"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.