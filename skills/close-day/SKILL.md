---
description: "Wrap up the workday with an evening routine. Use this skill when the user says 'close my day', 'end of day', 'wrap up', 'EOD', 'evening routine', or 'day summary'. Reviews work completed, catches up on communications, checks for unsent meeting summaries, and prepares for tomorrow."
---

# Close Day Skill

## Context
Closing the day properly ensures nothing falls through the cracks — work is documented, communications are handled, meeting follow-ups are sent, and tomorrow is set up for success. This skill orchestrates a thorough evening wrap-up.

## When to Use
- End of workday
- When the user wants to review and close out the day

## Workflow

### Step 0b: Extract Session Outcomes & Knowledge
Before reviewing the day's work, capture untracked work and reusable knowledge from today's Copilot sessions:

1. **Invoke the `session-outcomes` skill** with scope = today
2. This analyzes all sessions created or modified today, extracts distinct work items, and creates/links Daily Planner tasks
3. As part of outcome extraction, the `session-knowledge` skill runs to capture reusable knowledge — deployment steps, configurations, debugging patterns, architectural decisions — persisting them as Copilot instructions, skill updates, or Notion documentation
4. The extracted outcomes feed into Step 1's work summary — ensuring nothing is missed

> **Note:** This step runs automatically. If no sessions exist for today or all sessions have been processed already, it is skipped silently.

### Step 1: Review Day's Work
Pull what was accomplished and what's still in progress (now enriched with session outcomes from Step 0b):
```
DailyPlanner-get_tasks(status: "Completed", dueDate: "today")
DailyPlanner-get_tasks(status: "In Progress")
```

Present a work summary:
```markdown
## 📊 Today's Work Summary

### ✅ Completed
| Task | Type | Notes |
|------|------|-------|
| Fix auth bug | Engineering | PR #456 merged |
| Review API design | Review | Approved with comments |

### 🔄 In Progress
| Task | Started | What's Left |
|------|---------|-------------|
| Build email template | Today | Frontend done, need backend |

### 📈 Metrics
- Tasks completed: [N]
- Tasks started: [M]
- Meetings attended: [K]
```

### Step 2: Check Unsent Meeting Summaries
Review today's meetings and check if summary emails were sent:

```
DailyPlanner-get_todays_meetings()
```

For each meeting:
1. Check if notes exist in Notion (via `notion-API-post-search`)
2. Check if a summary email was sent (via activity log or WorkIQ)

Flag meetings without notes or summaries:
```markdown
## 📝 Meeting Follow-Up Check
| Meeting | Notes in Notion | Summary Sent | Action |
|---------|----------------|--------------|--------|
| Standup | ✅ | N/A (standup) | — |
| Design Review | ✅ | ❌ Not sent | Draft summary? |
| 1:1 Duncan | ❌ No notes | ❌ Not sent | Use `meeting-end` |
```

For meetings missing notes/summaries, offer:
- "Would you like me to capture notes now for [meeting]?" → invoke `meeting-end`
- "Would you like me to draft a summary email for [meeting]?" → invoke meeting-end Step 9

### Step 3: End-of-Day Email & Teams Check
Catch up on any communications that arrived during the afternoon:

1. **Unanswered emails:**
   ```
   workiq-ask_work_iq: "Do I have any emails from today that I haven't responded to that need a response? Summarize with sender, subject, and what they need."
   ```

2. **Teams messages:**
   ```
   workiq-ask_work_iq: "Are there any Teams messages or @mentions from today that I haven't addressed? Summarize."
   ```

3. **Action items from today's communications:**
   ```
   workiq-ask_work_iq: "Based on today's emails and Teams messages, are there any action items I should track? List each with who asked, what they need, and any deadline."
   ```

Present unanswered items:
```markdown
## 📬 Unanswered Communications
### Needs Response
| From | Subject/Channel | What They Need | Urgency |
|------|----------------|----------------|---------|
| [Sender] | [Subject] | [Brief summary] | [Today/This week] |

### New Action Items from Comms
| Item | Source | Requested By | Deadline |
|------|--------|-------------|----------|
| [Action] | Email | [Person] | [Date] |
```

For each new action item, offer to create a task in Daily Planner.

### Step 4: Check for Documentation Gaps
Run parallel checks across Daily Planner modules:

1. **Exercise:**
   ```
   DailyPlanner-get_exercises() — filter today
   ```
   Flag: "⚠️ No workout logged today" if empty.

2. **Diet/Nutrition:**
   ```
   DailyPlanner-get_diet_entries(date: "today yyyy-MM-dd")
   ```
   Flag: "⚠️ No meals logged today" if empty.

3. **Water Intake:**
   ```
   DailyPlanner-get_water_intake(date: "today yyyy-MM-dd")
   ```
   Flag: "⚠️ Below 2000ml target" if low.

4. **Financial:**
   ```
   DailyPlanner-get_expenses(from: "today", to: "today")
   ```
   Informational only.

Present gap report:
```markdown
## 📋 Day Completion Check
| Area | Status | Action |
|------|--------|--------|
| Tasks | ✅ 5 completed, 2 in progress | — |
| Meetings | ⚠️ 1 of 3 missing notes | Use `meeting-end` |
| Emails | ⚠️ 2 unanswered | Respond or defer |
| Exercise | ⚠️ Not logged | Log workout or skip |
| Nutrition | ✅ 3 meals logged | — |
| Water | ⚠️ 1500ml (below 2000ml) | Log more water |
| Finances | ℹ️ No expenses today | — |
```

### Step 5: Address Gaps (Interactive)
For each gap, ask the user if they want to:
- Log the missing data now
- Respond to the communication
- Defer to tomorrow
- Skip for today

### Step 6: Run Daily Journaling
Invoke the `daily-journal` skill to compile everything into a journal entry.
Pass the work summary, meeting outcomes, and communications data to enrich the journal.

### Step 7: Plan Tomorrow
Show what's coming tomorrow and offer full interactive planning:

1. **Quick preview:**
   ```
   DailyPlanner-get_todays_meetings(date: "tomorrow yyyy-MM-dd")
   DailyPlanner-get_tasks(dueDate: "tomorrow")
   ```

   Also check for early morning meetings:
   ```
   workiq-ask_work_iq: "What meetings do I have first thing tomorrow morning? Any that require preparation tonight?"
   ```

   Present:
   ```markdown
   ## 📆 Tomorrow Preview

   ### Meetings
   | Time | Meeting | Prep Needed? |
   |------|---------|-------------|
   | 9:00 AM | Sprint Planning | Yes — consider prep tonight |
   | 11:00 AM | Team Standup | No |

   ### Tasks Due
   | Task | Priority | Status |
   |------|----------|--------|
   | Submit design doc | 🔴 P1 | In Progress |

   ### Carried Forward from Today
   | Task | Today's Progress | Tomorrow Priority |
   |------|-----------------|-------------------|
   | Build email template | 60% done | Continue — P2 |

   ### ⚡ Early Action Required
   - [Meeting at 9 AM requires prep — consider doing it now or first thing tomorrow]
   ```

2. **Offer full planning:**
   ```
   ask_user: "Would you like to plan tomorrow in detail?"
     choices: ["Yes — run full day planning for tomorrow", "No — the preview is enough"]
   ```
   If yes: invoke `plan-day` skill with tomorrow's date. This gives full interactive time-blocking, task selection, and workout scheduling.

3. **Learning block suggestion:**
   Check if any learning topics are due for review or have low progress:
   ```
   DailyPlanner-get_learning_focus(limit: 3)
   ```
   If topics are available:
   ```
   📚 Learning suggestion: You have [N] topics due for study/review. Consider scheduling a 30-45 min learning block tomorrow.
   ```

### Step 8: Evening Tech Digest
Quick scan of what happened in tech today:

1. **Afternoon tech developments:**
   ```
   web_search: "technology news today highlights"
   web_search: "software engineering announcements today"
   ```

2. **Present digest** (brief, 3-5 items max):
   ```markdown
   ## 📰 Evening Tech Digest
   - **[Headline]** — [one-line summary and relevance]
   - **[Headline]** — [one-line summary]
   - **[Headline]** — [one-line summary]
   
   📖 Want to learn more about any of these? I can add them to your learning path.
   ```

3. **Learning connection:**
   If any news item relates to a topic in the user's learning path, highlight it:
   > "📚 This relates to your [subject/topic] — consider revisiting during your next learning session."

### Step 9: Carry-Over & Priority Setting
For in-progress tasks that didn't complete:
1. Ask if they should be marked for tomorrow's focus
2. Update `isToday` flag if confirmed
3. Suggest priority order for tomorrow

```markdown
## 🔄 Carry-Over to Tomorrow
| Task | Status | Carry Forward? |
|------|--------|---------------|
| Build email template | 60% done | ✅ Yes — marked for tomorrow |
| Review PR #789 | Not started | ✅ Yes — P2 priority |
```

### Step 10: End-of-Day Summary
Present a final summary:
```markdown
## 🌙 Day Closed — [Date]

### Accomplishments
- Completed [N] tasks
- Attended [M] meetings ([K] with notes captured)
- Reviewed [P] PRs
- Responded to [Q] emails

### Carried Forward
- [N] tasks moved to tomorrow
- [M] unanswered communications deferred

### Tomorrow's Focus
1. [Top priority for tomorrow]
2. [Second priority]
3. [First meeting to prep for]
```

## Tools & APIs Used
- `DailyPlanner-get_tasks` — Completed, in-progress, and tomorrow's tasks
- `DailyPlanner-get_todays_meetings` — Today's and tomorrow's meetings
- `DailyPlanner-get_exercises` — Workout check
- `DailyPlanner-get_diet_entries` — Nutrition check
- `DailyPlanner-get_water_intake` — Hydration check
- `DailyPlanner-get_expenses` — Financial check
- `DailyPlanner-update_task` — Carry-over tasks
- `notion-API-post-search` — Check meeting notes existence
- `workiq-ask_work_iq` — Unanswered emails, Teams messages, tomorrow's meetings
- `web_search` — Afternoon tech developments, software engineering announcements
- `daily-journal` skill — Compose and save journal
- `meeting-end` skill — Capture missed meeting notes
- `ask_user` — Interactive gap filling

## Output Format
Work summary → meeting follow-up check → communications catch-up → gap report → journal → tomorrow preview → carry-over → end-of-day summary.

## Notes
- Don't be pushy about health tracking gaps — some days won't have exercise or expenses
- Prioritize work-related gaps (unsent meeting summaries, unanswered emails) over health tracking
- The journal entry should capture the full picture of the day
- If the user skips journaling, still show the summary and tomorrow preview
- For Fridays, expand tomorrow preview to show Monday's schedule and flag weekend prep items
