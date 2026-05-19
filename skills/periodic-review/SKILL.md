---
description: "Run a periodic review of goals, tasks, health, learning, and finances. Use this skill when the user says 'daily review', 'weekly review', 'monthly review', 'review progress', 'how am I doing', 'goal check', or 'progress report'. Covers all areas tracked in the Daily Planner with progress indicators."
---

# Periodic Review Skill

## Context
Regular reviews ensure goals stay on track and nothing falls behind. This skill provides a comprehensive review across all areas of the Daily Planner — work, health, learning, and finances — at daily, weekly, or monthly granularity.

## When to Use
- Daily check-in on progress
- Weekly retrospective
- Monthly goal review
- When the user wants a holistic view of their progress

## Workflow

### Step 0: Determine Period
Detect the review period from the user's request:
- **Daily:** Today's data
- **Weekly:** Last 7 days
- **Monthly:** Last 30 days (or calendar month)

If unclear, ask:
```
ask_user: "What review period? Daily (today), Weekly (last 7 days), or Monthly (this month)?"
```

### Step 1: Goals Progress
```
DailyPlanner-get_goals(status: "In Progress")
```

For each goal, check:
- Current progress percentage
- Linked tasks completed vs remaining
- Due date proximity

### Step 2: Tasks Summary
```
DailyPlanner-get_tasks(status: "Completed")  — filter by period
DailyPlanner-get_tasks(status: "In Progress")
DailyPlanner-get_tasks(dueDate: "overdue")
```

Calculate:
- Tasks completed in period
- Tasks still in progress
- Overdue tasks
- Completion rate

### Step 3: Learning Progress
```
DailyPlanner-get_subjects(status: "Active")
DailyPlanner-get_learning_focus()
```

Show:
- Active subjects and their progress
- Topics worked on during the period
- Time invested in learning

### Step 4: Health & Wellness
```
DailyPlanner-get_exercises()  — filter by period
DailyPlanner-get_body_measurements(latestOnly: true)
DailyPlanner-get_water_intake()  — recent
DailyPlanner-get_diet_entries()  — recent
```

Summarize:
- Workout frequency and types
- Nutrition trends
- Hydration consistency
- Body measurement trends (weekly/monthly only)

### Step 5: Financial Health (if weekly/monthly)
```
DailyPlanner-get_budget_summary()
DailyPlanner-get_finance_dashboard(startDate: "[period start]", endDate: "[period end]")
```

Show:
- Income vs expenses for the period
- Budget envelope usage
- Savings rate

### Step 6: Journal Review (weekly/monthly)
```
DailyPlanner-get_journal_entries(fromDate: "[period start]", toDate: "[period end]")
```

Show:
- Mood trends
- Recurring themes
- Highlights

### Step 7: Compose Review Report

```markdown
# 📊 [Period] Review — [Date Range]

## 🎯 Goals
| Goal | Progress | Trend | Status |
|------|----------|-------|--------|
| [Goal 1] | ████████░░ 80% | ↑ +10% | On track |
| [Goal 2] | ███░░░░░░░ 30% | → 0% | ⚠️ Stalled |

## 📋 Tasks
- ✅ Completed: [X] tasks
- 🔄 In Progress: [X] tasks
- ⏰ Overdue: [X] tasks
- 📈 Completion rate: [X]%

## 📚 Learning
| Subject | Progress | Time Spent |
|---------|----------|------------|
| [Subject] | 60% | 5 hrs |

## 💪 Health & Wellness
- 🏃 Workouts: [X] sessions ([types])
- 💧 Avg water: [X] ml/day
- 🍽️ Meals logged: [X] days
- ⚖️ Weight: [current] kg (trend: [↑↓→])

## 💰 Finances (weekly/monthly)
- 📈 Income: KES [X]
- 📉 Expenses: KES [X]
- 💰 Net: KES [X]
- 📊 Budget usage: [X]%

## 💭 Mood & Reflections
- Predominant mood: [mood]
- Journal entries: [X]
- Key themes: [themes]

## 📌 Focus Areas for Next Period
1. [Recommendation based on data]
2. [Area needing attention]
3. [Opportunity to capitalize on]
```

## Tools & APIs Used
- `DailyPlanner-get_goals` — Goal progress
- `DailyPlanner-get_tasks` — Task metrics
- `DailyPlanner-get_subjects` / `get_learning_focus` — Learning
- `DailyPlanner-get_exercises` — Workouts
- `DailyPlanner-get_body_measurements` — Body metrics
- `DailyPlanner-get_water_intake` — Hydration
- `DailyPlanner-get_diet_entries` — Nutrition
- `DailyPlanner-get_budget_summary` / `get_finance_dashboard` — Finances
- `DailyPlanner-get_journal_entries` — Journals and mood

## Output Format
Multi-section review report with progress bars, trend indicators, and actionable recommendations.

## Notes
- Daily reviews are lighter (skip finances, body measurements, journals)
- Weekly reviews include all sections
- Monthly reviews add trend analysis and goal recalibration suggestions
- Always end with actionable focus areas for the next period

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
  description = "Surfaced by: periodic-review · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "periodic-review"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.