---
description: "Generate a weekly status report across all teams. Use this skill when the user says 'weekly status', 'status report', 'weekly rollup', 'team status', 'weekly update', or 'what did I do this week'. Aggregates completed tasks per team, PRs, meetings, and blockers into a single report."
---

# Weekly Status Report Skill

## Context
As an architect across 8 teams, Pius needs a consolidated weekly rollup showing work across all teams. This skill aggregates the week's work from Daily Planner and presents it organized by team.

## When to Use
- End of week (Friday or before weekly reports are due)
- When asked for a status update covering multiple teams
- When preparing for team-wide or org-wide meetings

## Workflow

### Step 1: Determine Week Range
Calculate the current work week (Monday–Friday) or use the user's specified range.

### Step 2: Get Completed Tasks
```
DailyPlanner-get_tasks(status: "Completed")
```
Filter to the week's date range.

### Step 3: Get In-Progress Work
```
DailyPlanner-get_tasks(status: "In Progress")
```

### Step 4: Get Meetings This Week
```
For each day of the week:
DailyPlanner-get_todays_meetings(date: "[day]")
```

### Step 5: Check Cross-Team Activity
```
workiq-ask_work_iq: "Summarize the key work items, PR reviews, and cross-team collaborations I was involved in this week."
```

### Step 6: Group by Team
Using `my-teams` skill context, organize all data by team tag:
- Reliability Data Engineering
- Benchmarking
- Anomaly Detection
- Sustainability
- Performance
- Gates & Defense
- COSINE
- Team Duma

### Step 7: Compose Weekly Report

```markdown
# 📊 Weekly Status Report — Week of [Date]

## Summary
- ✅ Tasks completed: [X]
- 🔄 Tasks in progress: [X]
- 📅 Meetings attended: [X]
- 🤝 Teams engaged: [X] of 8

---

## By Team

### 🔧 Reliability Data Engineering
**Completed:**
- [Task 1] — [brief outcome]
- [Task 2] — [brief outcome]

**In Progress:**
- [Task 3] — [progress/ETA]

**Meetings:** [X] meetings ([list])

---

### 📊 Benchmarking
**Completed:**
- [Task] — [outcome]

**In Progress:**
- [Task] — [progress]

---

[Repeat for each team with activity]

---

### Teams with No Activity This Week
- [Team X] — No tasks or meetings this week

---

## 🚧 Blockers & Risks
| Blocker | Team | Impact | Action |
|---------|------|--------|--------|
| [blocker] | [team] | [impact] | [next step] |

## 📌 Next Week Priorities
1. [Priority 1] — [team]
2. [Priority 2] — [team]
3. [Priority 3] — [team]
```

### Step 8: Offer Export
- Export as Word doc (markdown-to-word skill)
- Export as slides (md2pptx skill)
- Save to Notion
- Copy for email

## Tools & APIs Used
- `DailyPlanner-get_tasks` — Completed and in-progress tasks
- `DailyPlanner-get_todays_meetings` — Daily meetings
- `my-teams` skill — Team context and grouping
- `workiq-ask_work_iq` — Cross-team activity
- `markdown-to-word` / `md2pptx` — Export options
- `ask_user` — Confirm period, add context

## Output Format
Structured weekly report grouped by team with summary metrics, per-team sections, blockers, and next-week priorities.

## Notes
- Teams with no activity still get mentioned (transparency)
- Keep task descriptions outcome-focused, not activity-focused
- Blockers section is critical — always include even if empty
- This report can feed into org-wide rollups
