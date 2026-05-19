---
description: "Get a unified health and wellness dashboard. Use this skill when the user says 'health check', 'wellness check', 'fitness summary', 'how is my health', 'body check', 'workout summary', or 'health dashboard'. Combines exercise, diet, water intake, and body measurements into one view."
---

# Health Check Skill

## Context
Health tracking is spread across multiple Daily Planner modules — exercise, diet, water, and body measurements. This skill unifies everything into a single wellness dashboard.

## When to Use
- Daily health check-in
- Weekly wellness review
- When invoked by `periodic-review` or `close-day` skills
- When the user wants a holistic health view

## Workflow

### Step 1: Pull Health Data (Parallel)

1. **Exercises:**
   ```
   DailyPlanner-get_exercises()
   DailyPlanner-get_exercise_logs(startDate: "[period start]", endDate: "[period end]")
   ```

2. **Diet:**
   ```
   DailyPlanner-get_diet_entries(date: "[today or period]")
   ```

3. **Water:**
   ```
   DailyPlanner-get_water_intake(date: "[today]")
   ```

4. **Body Measurements:**
   ```
   DailyPlanner-get_body_measurements(latestOnly: true)
   ```

### Step 2: Compose Dashboard

```markdown
# 💪 Health & Wellness Dashboard — [Date]

## 🏃 Exercise
| Exercise | Duration | Sets × Reps | Calories | Status |
|----------|----------|-------------|----------|--------|
| Running | 30 min | — | 300 | ✅ Complete |
| Push-ups | 15 min | 3 × 20 | 100 | ✅ Complete |

**This week:** [X] workouts | **Streak:** [X] days

## 🍽️ Nutrition
| Meal | Food | Calories | Protein | Carbs | Fat |
|------|------|----------|---------|-------|-----|
| Breakfast | Oatmeal | 350 | 12g | 50g | 8g |
| Lunch | Chicken salad | 450 | 35g | 20g | 18g |

**Daily totals:** [X] cal | [X]g protein | [X]g carbs | [X]g fat

## 💧 Hydration
- Today: [X] ml / 2000 ml target
- Progress: ████████░░ [X]%
- Glasses: [X] / 8

## ⚖️ Body Measurements (Latest)
| Metric | Value | Trend |
|--------|-------|-------|
| Weight | [X] kg | [↑↓→] |
| Body Fat | [X]% | [↑↓→] |
| BMI | [X] | [↑↓→] |
| Waist | [X] cm | [↑↓→] |

## 💡 Recommendations
- [Based on data: "Consider logging water intake" or "Great workout consistency!"]
```

## Tools & APIs Used
- `DailyPlanner-get_exercises` / `get_exercise_logs` — Workouts
- `DailyPlanner-get_diet_entries` — Nutrition
- `DailyPlanner-get_water_intake` — Hydration
- `DailyPlanner-get_body_measurements` — Body metrics

## Output Format
Unified health dashboard with tables and progress indicators.

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
  description = "Surfaced by: health-check · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "health-check"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.