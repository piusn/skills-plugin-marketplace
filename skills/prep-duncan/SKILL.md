---
description: "Prepare for meetings or emails with Duncan (direct manager). Use this skill when the user says 'prepare for Duncan', 'Duncan meeting', '1:1 prep', 'Duncan update', 'Duncan email', 'manager meeting', or 'manager prep'. Summarizes all work streams in a presentable format covering tasks, teams, personal development, and blockers."
---

# Prep for Duncan (Direct Manager) Skill

## Context
Duncan is Pius's direct manager. 1:1s with Duncan cover all work — not just official/impact tasks, but also team support, personal development, learning, and day-to-day operational items. The update should be comprehensive but well-organized.

## When to Use
- Before 1:1 meetings with Duncan
- When composing status updates for Duncan
- Weekly check-in preparation

## Workflow

### Step 1: Get All Active Work
Pull the full picture of current work:

1. **All active tasks:**
   ```
   DailyPlanner-get_tasks(status: "In Progress")
   DailyPlanner-get_tasks(status: "New", priority: "P1")
   DailyPlanner-get_tasks(status: "New", priority: "P2")
   ```

2. **Recently completed:**
   ```
   DailyPlanner-get_tasks(status: "Completed")
   ```
   Filter to the last week (or since last 1:1).

3. **Blocked items:**
   ```
   DailyPlanner-search_tasks(query: "blocked")
   ```

### Step 2: Group by Team/Project
Using the `my-teams` skill context, organize tasks by team:
- Reliability Data Engineering
- Benchmarking
- Anomaly Detection
- Sustainability
- Performance
- Gates & Defense
- COSINE
- Team Duma
- Cross-team / Personal

### Step 3: Check Goals & Learning
Pull personal development items:

1. **Goals:**
   ```
   DailyPlanner-get_goals(status: "In Progress")
   ```

2. **Learning progress:**
   ```
   DailyPlanner-get_subjects(status: "Active")
   ```

### Step 4: Get Recent Context
Use WorkIQ for any relevant communications:
```
workiq-ask_work_iq: "Summarize my key work discussions, decisions, and any items Duncan mentioned or assigned to me in the past week."
```

### Step 5: Compose 1:1 Update
Format for a productive 1:1 conversation:

```markdown
# 1:1 Update for Duncan — [Date]

## 📊 Work Summary (Since Last 1:1)

### Completed ✅
| Task | Team | Notes |
|------|------|-------|
| [Task] | Reliability DE | [brief outcome] |
| [Task] | Benchmarking | [brief outcome] |

### In Progress 🔄
| Task | Team | Progress | ETA |
|------|------|----------|-----|
| [Task] | Anomaly Detection | 60% | Mar 20 |
| [Task] | Performance | 30% | Mar 25 |

### Blocked 🚧
| Task | Team | Blocker | Help Needed |
|------|------|---------|-------------|
| [Task] | [Team] | [blocker] | [ask] |

## 🎯 Goals Progress
| Goal | Progress | Status |
|------|----------|--------|
| [Goal 1] | ████████░░ 80% | On track |
| [Goal 2] | ███░░░░░░░ 30% | Needs attention |

## 📚 Personal Development
- Learning: [current subject/topic]
- Growth areas: [areas being developed]

## 💬 Discussion Points
1. [Topic to discuss]
2. [Decision needed]
3. [Feedback or support request]

## 📌 Next Week Focus
- [Priority 1]
- [Priority 2]
```

### Step 6: Offer Export Options
- Copy for email
- Export as Word doc
- Save to Notion

## Tools & APIs Used
- `DailyPlanner-get_tasks` — All task categories
- `DailyPlanner-search_tasks` — Blocked items
- `DailyPlanner-get_goals` — Goal progress
- `DailyPlanner-get_subjects` — Learning progress
- `my-teams` skill — Team grouping
- `workiq-ask_work_iq` — Recent communications
- `markdown-to-word` skill — Optional export
- `ask_user` — Add discussion points

## Output Format
Comprehensive 1:1 update organized by completed/in-progress/blocked, grouped by team, with goals, learning, and discussion points.

## Notes
- Duncan needs the full picture — include all work, not just official
- Group by team to show breadth of architect role
- Always include discussion points — come prepared with topics
- Personal development section shows investment in growth
- Keep task descriptions brief — focus on status and outcomes
