---
description: "Plan and organize a specific day by checking calendar, tasks, and priorities, then updating the Daily Planner. Use this skill when the user says 'plan my day', 'plan today', 'plan tomorrow', 'organize my day', 'plan [date]', 'schedule my day', 'what should I work on', or 'daily planning'. Pulls calendar from WorkIQ, compares with Daily Planner, identifies gaps, and interactively helps the user commit to a day plan."
---

# Plan My Day Skill

## Context
Planning a day means actively deciding what to work on and when, not just viewing what's there. This skill pulls the full picture (calendar, tasks, overdue items, goals), helps the user make decisions, and **writes the plan back to Daily Planner** — marking tasks for today, setting priorities, and aligning work with meetings.

Unlike `start-day` (a read-only morning dashboard), this skill is an **interactive planning session** that modifies the Daily Planner.

## When to Use
- Planning today, tomorrow, or any specific date
- Sunday evening to plan Monday
- After a disruption that requires replanning
- When the user feels overwhelmed and needs to prioritize

## Workflow

### Step 1: Determine the Date
The date can be dynamic. Detect from the user's request:
- "plan my day" → today
- "plan tomorrow" → tomorrow
- "plan Monday" → next Monday
- "plan March 15" → specific date

If unclear:
```
ask_user: "Which day do you want to plan?"
  choices: ["Today ([date])", "Tomorrow ([date])", "A specific date"]
```

### Step 2: Pull Calendar from WorkIQ
Get the official calendar for the target date:
```
workiq-ask_work_iq: "What meetings do I have on [target date formatted as 'Monday March 15, 2026']? For each meeting list: title, start time, end time, duration, location or Teams link, and key attendees."
```

### Step 3: Pull Daily Planner State
In parallel, get current planner data:

1. **Existing meetings on that date:**
   ```
   DailyPlanner-get_todays_meetings(date: "[yyyy-MM-dd]")
   ```

2. **Tasks already marked for today:**
   ```
   DailyPlanner-get_tasks(isToday: true)
   ```

3. **Overdue tasks:**
   ```
   DailyPlanner-get_tasks(dueDate: "overdue")
   ```

4. **Tasks due on target date:**
   ```
   DailyPlanner-get_tasks(dueDate: "today")
   ```
   (or filter by target date from all tasks)

5. **In-progress tasks:**
   ```
   DailyPlanner-get_tasks(status: "In Progress")
   ```

6. **Suggested focus:**
   ```
   DailyPlanner-get_suggested_focus()
   ```

### Step 4: Sync Calendar → Daily Planner
Compare WorkIQ meetings with Daily Planner meetings. For each missing meeting:
```
DailyPlanner-create_meeting(
  title: "[meeting title]",
  date: "[yyyy-MM-ddTHH:mm]",
  durationMinutes: [duration],
  location: "[location/link]",
  tags: "[team tag if identifiable]"
)
```

Report sync results:
```markdown
## 📅 Calendar Sync for [Date]
| Status | Meeting | Time |
|--------|---------|------|
| ✅ Synced | Standup | 9:00-9:30 |
| ➕ Added | Design Review | 14:00-15:00 |
| ⏭️ Already exists | 1:1 with Duncan | 11:00-11:30 |
```

### Step 5: Map Available Time
Calculate available work blocks around meetings:

```markdown
## ⏰ Available Time Blocks — [Date]

| Block | Time | Duration | Suggested Use |
|-------|------|----------|---------------|
| 🟢 Morning | 08:00 – 09:00 | 1 hr | Deep work |
| 📅 Meeting | 09:00 – 09:30 | 30 min | Standup |
| 🟢 Mid-morning | 09:30 – 11:00 | 1.5 hrs | Deep work |
| 📅 Meeting | 11:00 – 11:30 | 30 min | 1:1 with Duncan |
| 🟢 Pre-lunch | 11:30 – 12:30 | 1 hr | Tasks |
| 🍽️ Lunch | 12:30 – 13:30 | 1 hr | Break |
| 🟢 Afternoon | 13:30 – 14:00 | 30 min | Quick tasks |
| 📅 Meeting | 14:00 – 15:00 | 1 hr | Design Review |
| 🟢 Late afternoon | 15:00 – 17:00 | 2 hrs | Deep work |

**Total available:** ~6 hours of work time
```

### Step 6: Present Task Candidates
Show all tasks that could fill the available time, ranked by priority:

```markdown
## 📋 Task Candidates

### 🔴 Overdue (must address)
| # | Task | Priority | Due | Est. Time |
|---|------|----------|-----|-----------|
| 1 | [Task] | P1 | Mar 10 | ~2 hrs |
| 2 | [Task] | P2 | Mar 12 | ~1 hr |

### 🟠 Due Today
| # | Task | Priority | Status | Est. Time |
|---|------|----------|--------|-----------|
| 3 | [Task] | P2 | In Progress | ~1.5 hrs |

### 🔵 In Progress (momentum)
| # | Task | Priority | Progress | Est. Time |
|---|------|----------|----------|-----------|
| 4 | [Task] | P2 | 60% done | ~1 hr |

### 🟢 Suggested Focus (from goals)
| # | Task | Priority | Goal | Est. Time |
|---|------|----------|------|-----------|
| 5 | [Task] | P3 | Marathon Prep | ~30 min |
| 6 | [Task] | P3 | Career Growth | ~2 hrs |

### ⏳ Quick Wins (< 30 min)
| # | Task | Priority | Est. Time |
|---|------|----------|-----------|
| 7 | [Task] | P4 | ~15 min |
| 8 | [Task] | P4 | ~20 min |
```

### Step 7: Interactive Planning
Let the user select what goes into their day:

```
ask_user: "Which tasks do you want to commit to for [date]? Pick by number, or tell me your priorities and I'll suggest a plan."
  choices: ["Let me pick tasks by number", "Auto-plan based on priorities", "Mix — show me a suggested plan first"]
```

**If auto-plan:**
Apply this logic:
1. Overdue P1/P2 tasks fill the first available deep-work block
2. Due-today tasks get the next slots
3. In-progress tasks get remaining deep-work blocks (momentum)
4. Quick wins fill short gaps between meetings
5. Goal-aligned tasks fill any remaining time
6. Never schedule more than 6 hours of task work (leave buffer)

**If user picks:**
Accept task numbers and assign to time blocks.

### Step 8: Present the Day Plan
Show the final committed plan:

```markdown
## 📅 Day Plan — [Date]

| Time | Activity | Type | Task ID |
|------|----------|------|---------|
| 08:00 – 09:00 | Fix auth bug (P1, overdue) | 🔴 Deep work | #abc123 |
| 09:00 – 09:30 | Standup | 📅 Meeting | — |
| 09:30 – 11:00 | Design review doc (P2) | 🟠 Deep work | #def456 |
| 11:00 – 11:30 | 1:1 with Duncan | 📅 Meeting | — |
| 11:30 – 12:00 | Reply to PR comments (quick win) | ⚡ Quick task | #ghi789 |
| 12:00 – 12:30 | Update API docs (quick win) | ⚡ Quick task | #jkl012 |
| 12:30 – 13:30 | Lunch | 🍽️ Break | — |
| 13:30 – 14:00 | Prep for Design Review meeting | 📋 Meeting prep | — |
| 14:00 – 15:00 | Design Review | 📅 Meeting | — |
| 15:00 – 17:00 | Implement search feature (P2) | 🔵 Deep work | #mno345 |
| 17:00 – 17:30 | Close day / journaling | 📝 Wrap-up | — |

### Summary
- 🔴 Overdue addressed: 1
- 🟠 Due today: 1
- 🔵 In progress: 1
- ⚡ Quick wins: 2
- 📅 Meetings: 3
- ⏱️ Planned work: 5.5 hours
- 💡 Buffer: 0.5 hours
```

```
ask_user: "Does this plan look good? Want to adjust anything?"
```

### Step 9: Write Plan to Daily Planner
Once confirmed, update the Daily Planner:

**Mark selected tasks for today:**
For each task in the plan:
```
DailyPlanner-update_task(taskId: "[task_id]", isToday: true)
```

**Un-mark tasks NOT selected** (if previously marked for today but not in the plan):
```
DailyPlanner-update_task(taskId: "[task_id]", isToday: false)
```

**Reprioritize if needed:**
If the user changed any priorities during planning:
```
DailyPlanner-update_task(taskId: "[task_id]", priority: "[new priority]")
```

### Step 10: Meeting Prep Reminders
For each meeting on the planned day, check if prep is needed:

```markdown
## 📋 Meeting Prep Reminders

| Meeting | Time | Prep Needed | Action |
|---------|------|-------------|--------|
| 1:1 with Duncan | 11:00 | ⚠️ Yes | Run `prep-duncan` before 10:45 |
| Design Review | 14:00 | ⚠️ Yes | Run `meeting-prep Design Review` by 13:30 |
| Standup | 09:00 | ✅ No prep | — |
```

### Step 11: Health & Wellness Slot
Suggest a workout slot if none is planned:
```markdown
## 💪 Wellness Check

| Activity | Status | Suggestion |
|----------|--------|------------|
| Workout | ⚠️ Not planned | Morning (06:30) or evening (17:30)? |
| Water | — | Set 2000ml target |
| Meals | — | Plan lunch timing around meetings |
```

```
ask_user: "Want to schedule a workout for [date]?"
  choices: ["Yes — morning (before 08:00)", "Yes — evening (after 17:00)", "Skip for today"]
```

If yes, create the exercise:
```
DailyPlanner-create_exercise(
  type: "[from workout-prep suggestion]",
  date: "[yyyy-MM-dd]",
  status: "Planned",
  timeOfDay: "[Morning/Evening]"
)
```

### Step 12: Learning Block

Check for learning topics that need attention and offer to schedule a study block:

1. **Check learning focus:**
   ```
   DailyPlanner-get_learning_focus(limit: 5)
   ```

2. **Check revision queue:**
   Look for topics due for spaced repetition review.

3. **Present learning candidates:**
   ```markdown
   ## 📚 Learning Block

   | # | Subject | Topic | Progress | Next Block | Duration |
   |---|---------|-------|----------|-----------|----------|
   | 1 | Kubernetes | Networking | 25% | Hands-on lab | 45 min |
   | 2 | CosmosDB | Data Modeling | 10% | Core concepts | 30 min |
   | 3 | Event-Driven | Kafka basics | 0% | Fundamentals | 30 min |

   ### 📅 Revision Due
   | Topic | Last Studied | Review Type | Duration |
   |-------|-------------|------------|----------|
   | Docker volumes | 7 days ago | Practice exercise | 25 min |
   ```

4. **Schedule if user confirms:**
   ```
   ask_user: "Schedule a learning block for [date]?"
     choices: ["Yes — 30 min block", "Yes — 45 min block", "Yes — 1 hour block", "Skip learning today"]
   ```
   If yes, assign to an available time slot in the day plan and mark the learning task.

### Step 13: Final Confirmation

```markdown
## ✅ Day Planned — [Date]

| What | Count |
|------|-------|
| Tasks marked for today | [X] |
| Meetings synced | [X] |
| Work hours planned | [X] hrs |
| Workout scheduled | ✅/❌ |

**Your top 3 priorities:**
1. 🔴 [Most important task]
2. 🟠 [Second priority]
3. 🔵 [Third priority]

💡 *Start with: "start task [first task id]"*
```

## Tools & APIs Used
- `workiq-ask_work_iq` — Pull official calendar for any date
- `DailyPlanner-get_todays_meetings` — Existing planner meetings
- `DailyPlanner-create_meeting` — Sync new meetings
- `DailyPlanner-get_tasks` — Tasks by status, priority, due date, today flag
- `DailyPlanner-get_suggested_focus` — AI-ranked priorities
- `DailyPlanner-update_task` — Mark tasks for today, update priorities
- `DailyPlanner-create_exercise` — Schedule workouts
- `ask_user` — Task selection, plan confirmation, adjustments

## Output Format
Calendar sync → available time blocks → task candidates → interactive selection → committed day plan → Daily Planner updates → prep reminders.

## How This Differs from `start-day`
| Aspect | `start-day` | `plan-day` |
|--------|-------------|------------|
| Purpose | Morning briefing | Interactive planning |
| Modifies planner? | No (read-only) | **Yes** (marks tasks, syncs meetings) |
| Date | Today only | **Any date** |
| Interaction | Show dashboard | **User selects tasks, confirms plan** |
| Time blocking | No | **Maps tasks to available time slots** |
| Workout planning | No | **Suggests and schedules exercise** |

## Notes
- Always sync calendar first — meetings are fixed, tasks fill the gaps
- Leave 30 min buffer — never plan 100% of available time
- Quick wins between meetings prevent context-switch waste
- Overdue items should always be surfaced prominently
- The plan should be realistic — 5-6 hours of focused task work is a productive day
- Re-planning is fine — run this skill again if the day changes
