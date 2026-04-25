---
description: "Start the workday with a comprehensive morning routine. Use this skill when the user says 'start my day', 'morning routine', 'good morning', 'what's on today', 'daily briefing', or 'morning dashboard'. Syncs calendar, catches up on emails/Teams, identifies action items, and presents a prioritized dashboard."
---

# Start Day Skill

## Context
Starting the day right means having a complete picture: what happened overnight, what needs attention now, and what's coming today. This skill orchestrates a thorough morning catch-up — syncing the calendar, scanning emails and Teams for action items, checking task status, and presenting everything in a prioritized dashboard.

## When to Use
- First thing in the morning
- When returning from a break or PTO and needing to catch up

## Workflow

### Step 0b: Yesterday's Session Catch-Up
Check if yesterday's Copilot sessions were fully processed for outcomes:

1. **Invoke the `session-outcomes` skill** with scope = yesterday
2. This catches any work done in sessions that wasn't captured during close-day (e.g., late-night sessions, sessions where close-day wasn't run)
3. Only processes sessions not already tracked — idempotent by design

> **Note:** This step runs automatically. If yesterday's sessions are already processed or no sessions exist, it is skipped silently.

### Step 1: Sync Calendar
Invoke the `sync-meetings` skill to pull meetings from the official calendar into Daily Planner.

### Step 2: Get Today's Tasks & Meetings
Pull focus items in parallel:

1. **Suggested Focus Tasks:**
   ```
   DailyPlanner-get_suggested_focus()
   ```

2. **Today's Tasks:**
   ```
   DailyPlanner-get_tasks(isToday: true)
   ```

3. **Overdue Tasks:**
   ```
   DailyPlanner-get_tasks(dueDate: "overdue")
   ```

4. **Today's Meetings (now synced):**
   ```
   DailyPlanner-get_todays_meetings()
   ```

### Step 3: Email & Teams Catch-Up
Use WorkIQ to do a thorough scan of overnight/recent communications:

1. **Unread emails requiring action:**
   ```
   workiq-ask_work_iq: "What unread emails do I have that require a response or action? Summarize each with: sender, subject, what they need from me, and urgency level."
   ```

2. **Teams messages and mentions:**
   ```
   workiq-ask_work_iq: "What Teams messages or @mentions do I have from the last 24 hours that need my attention? Include channel name, sender, and what they're asking."
   ```

3. **Missed meeting follow-ups:**
   ```
   workiq-ask_work_iq: "Were there any meetings yesterday that I attended where follow-up emails were sent? Summarize any action items assigned to me."
   ```

4. **Documents shared with me:**
   ```
   workiq-ask_work_iq: "Have any documents, presentations, or files been shared with me in the last 24 hours? List with sender and brief description."
   ```

### Step 4: Extract Action Items
From the email/Teams catch-up, extract concrete action items:

1. **Categorize by urgency:**
   - 🔴 **Urgent** — needs response today (explicit deadlines, escalations, blocked teams)
   - 🟡 **Important** — should respond within 1-2 days (requests, reviews, feedback)
   - 🟢 **FYI** — informational, no action needed (status updates, announcements)

2. **Cross-reference with existing tasks:**
   Check if any extracted action items already exist in Daily Planner. If not, suggest creating them.

3. **Meeting-related action items:**
   For each meeting today, check if there are pending action items from previous meetings that should be discussed.

### Step 5: Meeting Prep Hints
For each meeting today, provide a quick context hint:

1. **Check Notion** for previous meeting notes:
   ```
   notion-API-post-search(query: "[meeting title]")
   ```

2. **For each meeting, show:**
   - Whether previous notes exist in Notion (✅ / ❌)
   - Number of pending action items related to this meeting
   - Whether a meeting prep has been done (suggest `meeting prep [title]` if not)
   - Time until meeting starts

### Step 6: PR & Code Review Check
Check for pending code reviews and PR activity:

1. **PRs needing my review:**
   ```
   workiq-ask_work_iq: "Do I have any pending pull request reviews or code review requests?"
   ```

2. **My open PRs:**
   Check for PRs the user has open that may have new comments or approvals.

### Step 7: Tech & Global News Briefing
Scan for relevant news to start the day informed:

1. **Technology news:**
   ```
   web_search: "top technology news today software engineering"
   web_search: "AI and machine learning news today"
   web_search: "cloud computing Azure Kubernetes news this week"
   ```

2. **Global markets & trends:**
   ```
   web_search: "global technology market news today"
   web_search: "trending technology news today"
   ```

3. **Summarize top stories** (pick 5-7 most relevant):
   Focus on stories relevant to a software architect:
   - AI/ML advancements and new tools
   - Cloud platform updates (Azure, AWS, GCP)
   - Open-source project releases
   - Security vulnerabilities and patches
   - Industry trends and shifts
   - Market/funding news for developer tools

### Step 8: Present Morning Dashboard
Format as a clean, scannable dashboard:

```markdown
# ☀️ Good Morning — [Date]

## 📬 Overnight Catch-Up
### 🔴 Urgent (respond today)
| From | Subject/Channel | What They Need | Action |
|------|----------------|----------------|--------|
| [Sender] | [Subject] | [Brief summary] | Reply / Review / Approve |

### 🟡 Important (respond soon)
| From | Subject/Channel | What They Need |
|------|----------------|----------------|
| [Sender] | [Subject] | [Brief summary] |

### 🟢 FYI (no action needed)
- [Summary of informational items]

### 📄 Documents Shared
- [Doc title] from [sender] — [brief description]

## 📅 Today's Meetings
| Time | Meeting | Prep Status | Pending Items | Action |
|------|---------|-------------|---------------|--------|
| 9:00 AM | Standup | ❌ No prep | 2 action items | `meeting prep Standup` |
| 2:00 PM | Design Review | ✅ Notes exist | 0 items | — |
| 4:00 PM | 1:1 with Duncan | ❌ No prep | 1 overdue item | `meeting prep Duncan` |

## 🎯 Focus Tasks (Top 5)
| Priority | Task | Due | Status |
|----------|------|-----|--------|
| 🔴 P1 | Fix auth bug | Today | In Progress |
| 🟠 P2 | Design review doc | Tomorrow | New |

## ⏰ Overdue Items
| Task | Due Date | Priority | Days Overdue |
|------|----------|----------|-------------|
| Update API docs | Mar 10 | P3 | 9 days |

## 🔍 Code Reviews
| PR | Repo | Author | Status |
|----|------|--------|--------|
| #456 Add caching | service-api | @John | Needs my review |
| #452 My PR: Fix timeout | service-api | Me | 2 approvals, ready to merge |

## 📌 Suggested Actions
1. [Most urgent email action]
2. [Overdue task to address]
3. [Meeting prep to run]
4. [PR to review]
5. [Task to start]

## 📰 Today's Tech Briefing
| # | Story | Category | Why It Matters |
|---|-------|----------|---------------|
| 1 | [Headline] | AI/ML | [Brief relevance to your work] |
| 2 | [Headline] | Cloud | [Brief relevance] |
| 3 | [Headline] | Security | [Brief relevance] |
| 4 | [Headline] | Open Source | [Brief relevance] |
| 5 | [Headline] | Market | [Brief relevance] |

💡 **Deep dive:** Ask me about any story for more details.

## 💡 Quick Commands
- `start task [id]` — Begin working on a task
- `meeting prep [name]` — Prepare for a meeting
- `plan my day` — Organize today's schedule
```

### Step 9: Set Day Context
Log the morning briefing:
```
DailyPlanner-add_activity_log(
  taskId: [first focus task],
  description: "Morning briefing completed. [N] urgent items, [M] meetings, [K] tasks due today."
)
```

## Tools & APIs Used
- `sync-meetings` skill — Calendar sync
- `DailyPlanner-get_suggested_focus` — Priority ranking
- `DailyPlanner-get_tasks` — Today's and overdue tasks
- `DailyPlanner-get_todays_meetings` — Meetings
- `workiq-ask_work_iq` — Emails, Teams messages, documents, meeting follow-ups, PR reviews
- `notion-API-post-search` — Previous meeting notes check
- `web_search` — Tech news, AI/ML updates, cloud platform news, market trends
- `DailyPlanner-add_activity_log` — Log briefing

## Output Format
Structured morning dashboard with:
- Overnight communications categorized by urgency (🔴/🟡/🟢)
- Meeting schedule with prep status and pending items
- Focus tasks and overdue items
- Code review queue
- Prioritized action list

## Notes
- Run all WorkIQ queries in parallel for speed — don't wait for each one sequentially
- If WorkIQ is unavailable, show tasks and meetings from Daily Planner only with a warning
- If no overdue items exist, skip that section (keep the dashboard clean)
- The "Suggested Actions" list should be ordered by true priority, not just data source
- For days after PTO/long breaks, expand the catch-up window to 3-5 days
