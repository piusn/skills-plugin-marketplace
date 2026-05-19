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

### Step 1: Sync Calendar
Invoke the `sync-meetings` skill to pull meetings from the official calendar into Daily Planner.

### Step 1.5: Sync Copilot Sessions
Sync yesterday's sessions that may not have been captured (covers sessions from after close-day or from other machines):

```
DailyPlanner-sync_copilot_sessions(since: "yesterday yyyy-MM-dd")
```

Brief report: "[N] sessions synced from yesterday"

This is a quick, non-interactive step. Just report the count and continue.

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

5. **Pull request notification emails:**
   ```
   workiq-ask_work_iq: "What pull request notification emails have I received in the last 24 hours? Include PR title, repository, author, what action is needed from me (review requested, comments added, approved, merged, changes requested), and any deadlines or urgency signals."
   ```

6. **Azure spend / cost alert emails:**
   ```
   workiq-ask_work_iq: "Do I have any emails about Azure spend, Azure cost alerts, budget warnings, or subscription cost reports from the last 48 hours? Summarize each with: which subscription or resource group, current spend vs budget, trend direction, and which team owns it."
   ```

7. **S360 SFI (Secure Future Initiative) items:**
   ```
   workiq-ask_work_iq: "Do I have any emails, Teams messages, or notifications about S360 items, Secure Future Initiative (SFI), security compliance, or ServiceTree security findings from the last 48 hours? Summarize each with: item title, severity, affected service or team, due date, and current status."
   ```

8. **S360 SFI follow-up from Teams channels:**
   ```
   workiq-ask_work_iq: "Are there any Teams messages in my channels about S360, SFI compliance, security bugs, or service health reviews that mention action items or deadlines? Include channel name, who posted, and what's needed."
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
Check for pending code reviews and PR activity from multiple sources:

1. **PRs needing my review (from email notifications):**
   Cross-reference PR notification emails gathered in Step 3 (item 5) to build a complete picture:
   - PRs where review was requested from me
   - PRs with new comments awaiting my response
   - PRs that are approved and ready to merge (my authored PRs)

2. **PRs needing my review (from GitHub):**
   ```
   workiq-ask_work_iq: "Do I have any pending pull request reviews or code review requests?"
   ```

3. **My open PRs with activity:**
   Check for PRs the user has open that may have new comments or approvals.

4. **Consolidate PR action items:**
   Merge results from email notifications and direct PR queries. Deduplicate and categorize:
   - 🔴 **Review requested** — someone is waiting on me
   - 🟡 **Comments to address** — feedback on my PRs
   - 🟢 **Ready to merge** — my PRs with sufficient approvals
   - ℹ️ **FYI** — merged/closed notifications (no action needed)

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

## 🔍 Code Reviews (from emails + GitHub)
| PR | Repo | Author | Action Needed | Source |
|----|------|--------|---------------|--------|
| #456 Add caching | service-api | @John | 🔴 Review requested | Email + GitHub |
| #452 My PR: Fix timeout | service-api | Me | 🟢 Ready to merge (2 approvals) | Email |
| #461 Update config | shared-lib | @Sarah | 🟡 Comments to address | Email |

## 💰 Azure Spend Review
| Subscription / Resource Group | Current Spend | Budget | Trend | Team | Action |
|-------------------------------|--------------|--------|-------|------|--------|
| RDE-Production | $12,400 | $15,000 | ↗️ +8% | Reliability DE | ⚠️ Follow up |
| Analytics-Dev | $3,200 | $5,000 | ↘️ -3% | Data Analytics | ✅ On track |

**Follow-ups needed:**
- [ ] Message [team] about [subscription] spend trending above budget
- [ ] Review [resource group] for unused resources

## 🔒 S360 / SFI Compliance
| Item | Severity | Service / Team | Due Date | Status | Action |
|------|----------|---------------|----------|--------|--------|
| [SFI finding title] | High | [Service] / [Team] | [Date] | Open | Follow up with team |
| [Security item title] | Medium | [Service] / [Team] | [Date] | In Progress | Check status |

**Follow-ups needed:**
- [ ] Message [team] about [SFI item] due [date]
- [ ] Verify [team] has started remediation for [item]

## 📌 Suggested Actions
1. [Most urgent email action]
2. [Azure spend follow-up if any budget alerts]
3. [S360/SFI item follow-up if any are due soon]
4. [PR review to complete]
5. [Overdue task to address]
6. [Meeting prep to run]
7. [Task to start]

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
- `workiq-ask_work_iq` — Emails, Teams messages, documents, meeting follow-ups, PR notifications, Azure spend alerts, S360/SFI items
- `notion-API-post-search` — Previous meeting notes check
- `web_search` — Tech news, AI/ML updates, cloud platform news, market trends
- `DailyPlanner-add_activity_log` — Log briefing

## Output Format
Structured morning dashboard with:
- Overnight communications categorized by urgency (🔴/🟡/🟢)
- Meeting schedule with prep status and pending items
- Focus tasks and overdue items
- Code review queue (merged from email notifications + GitHub)
- Azure spend review with team follow-ups
- S360/SFI compliance items with team follow-ups
- Prioritized action list

## Notes
- Run all WorkIQ queries in parallel for speed — don't wait for each one sequentially
- If WorkIQ is unavailable, show tasks and meetings from Daily Planner only with a warning
- If no overdue items exist, skip that section (keep the dashboard clean)
- The "Suggested Actions" list should be ordered by true priority, not just data source
- For days after PTO/long breaks, expand the catch-up window to 3-5 days

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
  description = "Surfaced by: start-day · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "start-day"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.