---
description: "Review learning progress across all subjects and topics with gap analysis. Use this skill when the user says 'learning review', 'study progress', 'how is my learning', 'what have I learned', 'learning dashboard', 'study report', 'learning gaps', or 'knowledge check'. Shows progress, identifies gaps, and recommends what to study next."
---

# Learning Review Skill

## Context
With 17 subjects and 32 topics in Daily Planner, plus notes in Notion, a regular review helps identify what's progressing, what's stalled, and where to focus next. This skill provides a comprehensive view across all learning areas.

## When to Use
- Weekly/monthly learning check-in
- When deciding what to study next
- When preparing for career development discussions
- When invoked by `periodic-review` skill

## Workflow

### Step 1: Pull All Learning Data

In parallel:

1. **All subjects:**
   ```
   DailyPlanner-get_subjects(status: "Active")
   ```

2. **All topics:**
   ```
   DailyPlanner-get_topics()
   ```

3. **All resources:**
   ```
   DailyPlanner-get_resources()
   ```

4. **Learning focus suggestions:**
   ```
   DailyPlanner-get_learning_focus(limit: 10)
   ```

### Step 2: Calculate Metrics

For each subject, calculate:
- **Topic count:** Total topics under this subject
- **Avg topic progress:** Average of all topic progress percentages
- **Resources completed:** Count of completed resources / total resources
- **Time invested:** Total timeSpent across resources (from progress updates)
- **Last activity:** Most recent progress update date
- **Staleness:** Days since last activity

### Step 3: Subject Dashboard

```markdown
# 📊 Learning Dashboard

## Overview
| Metric | Value |
|--------|-------|
| Active subjects | [X] |
| Total topics | [X] |
| Topics completed | [X] |
| Topics in progress | [X] |
| Topics not started | [X] |
| Resources completed | [X] / [X] |
| Total study time | [X] hours |

## 📚 Subject Progress

### 🔴 Needs Attention (0-25% or stale >30 days)
| Subject | Topics | Progress | Last Active | Status |
|---------|--------|----------|-------------|--------|
| Kubernetes | 4/8 started | ██░░░░░░░░ 15% | 45 days ago | 🔴 Stale |
| Security | 0/3 started | ░░░░░░░░░░ 0% | Never | 🔴 Not started |

### 🟡 In Progress (25-75%)
| Subject | Topics | Progress | Last Active | Status |
|---------|--------|----------|-------------|--------|
| .NET Aspire | 2/3 started | █████░░░░░ 45% | 5 days ago | 🟡 Active |
| Data Engineering | 3/4 started | ██████░░░░ 55% | 2 days ago | 🟡 Active |

### 🟢 Advanced (75%+)
| Subject | Topics | Progress | Last Active | Status |
|---------|--------|----------|-------------|--------|
| C# | 6/6 started | █████████░ 85% | 1 day ago | 🟢 Nearly done |

### ⏸️ Not Started
| Subject | Topics | Created | Priority |
|---------|--------|---------|----------|
| Machine Learning | 5 topics | Dec 2025 | Consider starting |
| Economics | 3 topics | Nov 2025 | Low priority |
```

### Step 4: Topic-Level Detail (Per Subject)

For subjects the user wants to drill into:

```markdown
## 📖 Topic Detail: [Subject Name]

| # | Topic | Progress | Resources | Time | Status |
|---|-------|----------|-----------|------|--------|
| 1 | [Topic 1] | ██████████ 100% | 3/3 ✅ | 4.5 hrs | Complete |
| 2 | [Topic 2] | █████░░░░░ 50% | 1/2 | 2.0 hrs | In Progress |
| 3 | [Topic 3] | ░░░░░░░░░░ 0% | 0/3 | 0 hrs | Not Started |
| 4 | [Topic 4] | ██░░░░░░░░ 20% | 1/4 | 1.5 hrs | In Progress |
```

### Step 5: Check Notion Notes Quality
For in-progress topics, check if notes exist in Notion:

```
notion-API-post-search(query: "[topic name]")
```

Flag topics with progress but no notes:
```
⚠️ [Topic X] is at 40% but has NO notes in Notion — consider reviewing and documenting key concepts
```

### Step 6: Gap Analysis

```markdown
## 🔍 Gap Analysis

### Knowledge Gaps (topics with resources but low retention)
| Topic | Progress | Notes | Concern |
|-------|----------|-------|---------|
| [Topic] | 60% | 0 notes | High progress, no notes — retention risk |
| [Topic] | 80% | 2 notes, 5 open questions | Many unresolved questions |

### Breadth Gaps (subjects with no activity)
- **Machine Learning** — 0% across all topics. Relevant to career goal?
- **Economics** — 0%. Consider archiving if not prioritized.

### Depth Gaps (subjects with shallow coverage)
- **Software Architecture** — 4 topics at 10-20%. Broad but shallow — consider deep-diving one topic at a time.

### Career Alignment
Cross-reference with career goals:
```
DailyPlanner-get_goals(tag: "Career")
```

| Goal | Relevant Subjects | Status |
|------|-------------------|--------|
| Principal SWE Promotion | Software Architecture, .NET, Azure | ⚠️ Needs more progress |
| AI/ML literacy | AI, Machine Learning, Data Science | 🔴 Not started |
```

### Step 6b: Technology Trend Scan
Proactively scan for new technologies and trends relevant to your role:

1. **Industry trends:**
   ```
   web_search: "top software architecture trends [current year]"
   web_search: "emerging technologies for platform engineers [current year]"
   web_search: "AI and machine learning engineering trends [current year]"
   ```

2. **Cloud and infrastructure trends:**
   ```
   web_search: "Azure new services and features [current quarter]"
   web_search: "Kubernetes ecosystem new tools [current year]"
   ```

3. **AI/ML coding trends:**
   ```
   web_search: "AI coding assistants and tools for developers [current year]"
   web_search: "LLM application development trends"
   ```

4. **Present trend summary:**
   ```markdown
   ## 🌐 Technology Trends
   
   ### 🔥 Hot Right Now
   | Technology/Trend | Category | Relevance to You | Action |
   |-----------------|----------|-------------------|--------|
   | [trend] | [AI/Cloud/Data/etc.] | [how it relates to your teams/work] | [Learn/Watch/Ignore] |

   ### 📈 Rising
   | Technology | Why It Matters | Related To Your Subjects |
   |-----------|---------------|-------------------------|
   | [tech] | [rationale] | [existing subject/topic connections] |

   ### 💡 Suggested Additions to Learning Path
   | Technology | Why Learn It | Priority | Prerequisite Subjects |
   |-----------|-------------|----------|----------------------|
   | [tech] | [rationale] | [High/Medium] | [what you need first] |
   ```

### Step 7: Recommendations

```markdown
## 📌 Recommendations

### This Week's Study Focus
Based on gap analysis, career alignment, and staleness:

1. 🔴 **[Subject/Topic]** — [reason: career-critical, stale, etc.]
2. 🟡 **[Subject/Topic]** — [reason: close to completion, momentum]
3. 🟢 **[Subject/Topic]** — [reason: quick win, finish it]

### Suggested Study Schedule
| Day | Focus | Duration | Topic |
|-----|-------|----------|-------|
| Mon | Deep work | 60 min | [Career-critical topic] |
| Wed | Breadth | 30 min | [New topic exploration] |
| Fri | Review | 30 min | [Revisit stale topic] |

### Actions
- 🗑️ **Archive:** [Subjects no longer relevant]
- 📝 **Add notes:** [Topics with progress but no notes]
- 🎯 **Prioritize:** [Topics aligned to career goals]
- 📚 **Add resources:** [Topics with no resources]
```

### Step 7b: Cross-Subject Connection Analysis
Analyze how your learning subjects interconnect and identify synthesis opportunities:

1. **Connection map:**
   ```markdown
   ## 🔗 Learning Connections
   
   ### Strong Connections (learn together)
   | Subject A | Subject B | Connection | Project Opportunity |
   |-----------|-----------|-----------|-------------------|
   | Kubernetes | Docker | Container orchestration builds on containers | Deploy multi-service app |
   | Event-Driven | Kafka | Kafka is the primary implementation | Build event sourcing system |
   
   ### Emerging Connections (explore)
   | Subject A | Subject B | Potential Link |
   |-----------|-----------|---------------|
   | ML/AI | Data Engineering | ML pipelines need data infrastructure |
   | Security | Cloud Architecture | Zero-trust requires cloud-native security |
   ```

2. **Breadth vs. Depth assessment:**
   ```markdown
   ## 📊 Architect's Knowledge Map
   
   ### Breadth Coverage (should know basics of all)
   | Domain | Subjects Covered | Coverage |
   |--------|-----------------|----------|
   | Cloud & Infrastructure | [list] | [X/Y topics] |
   | Data & Storage | [list] | [X/Y topics] |
   | Security | [list] | [X/Y topics] |
   | AI/ML | [list] | [X/Y topics] |
   | Frontend | [list] | [X/Y topics] |
   | Backend & APIs | [list] | [X/Y topics] |
   | DevOps & CI/CD | [list] | [X/Y topics] |
   | Architecture Patterns | [list] | [X/Y topics] |
   
   ### Depth Focus (should have expert knowledge)
   | Domain | Depth Level | Target Level | Gap |
   |--------|------------|-------------|-----|
   | [domain] | 🟡 Intermediate | 🟢 Expert | [specific topics to deepen] |
   ```

3. **Integration project suggestions:**
   Based on current progress across subjects, suggest projects that combine multiple areas:
   ```markdown
   ## 🛠️ Suggested Integration Projects
   | Project | Topics Combined | Your Readiness | Estimated Time |
   |---------|----------------|---------------|---------------|
   | [project] | [topic A + B + C] | [Ready/Need topic X first] | [hours] |
   ```

### Step 8: Spaced Review Suggestions
For completed topics, suggest review based on last activity:

```markdown
## 🔄 Spaced Review (Completed topics due for review)

| Topic | Completed | Last Reviewed | Review Due |
|-------|-----------|---------------|------------|
| [Topic] | Jan 15 | Feb 10 | ⚠️ Mar 10 (overdue) |
| [Topic] | Feb 20 | Mar 5 | Mar 19 |

💡 *Spaced repetition: Review at 1 day → 3 days → 7 days → 14 days → 30 days → 90 days*
```

### Active Revision Queue
Topics due for spaced repetition review:

```markdown
## 📅 Revision Schedule

### Overdue Reviews
| Topic | Last Studied | Review Type | Days Overdue |
|-------|-------------|------------|-------------|
| [topic] | [date] | Quick review (15 min) | [N] days |

### Due This Week
| Topic | Due Date | Review Type | Suggested Activity |
|-------|----------|------------|-------------------|
| [topic] | [date] | Practice exercise | Redo mini-project |
| [topic] | [date] | Teach-back | Explain to a colleague or write a blog post |

### Coming Up
| Topic | Due Date | Review Type |
|-------|----------|------------|
| [topic] | [date] | Deep review |
```

Revision types by interval:
- **Day 3:** Quick review — revisit key concepts, check questions
- **Day 7:** Practice exercise — redo hands-on work from memory
- **Day 14:** Teach-back — explain the topic as if teaching someone
- **Day 30:** Integration — use with other topics in a mini-project
- **Day 90:** Deep review — read advanced material, update notes, reassess understanding

### Step 9: Convert Gaps to Action Items

After presenting the gap analysis and recommendations, offer to create actionable tasks:

1. **For knowledge gaps (topics not started or weak):**
   ```
   ask_user: "Create study tasks for these gaps?"
   ```
   If yes, for each gap:
   ```
   DailyPlanner-create_task(
     title: "Study: [topic name]",
     type: "Task",
     priority: "P3",
     tags: ["Learning", "[subject name]"],
     description: "Gap identified in learning review. Focus: [gap description]. Subject: [subject]. Recommended resources: [resources if known]."
   )
   ```

2. **For breadth gaps (entire domains not covered):**
   Suggest running `learning-setup` to create a new learning path:
   ```
   💡 Domain gap: [domain] has no learning path. Run `learning-setup` to create one?
   ```

3. **For revision-due topics:**
   Create revision tasks:
   ```
   DailyPlanner-create_task(
     title: "Review: [topic name]",
     type: "Task",
     priority: "P4",
     tags: ["Learning", "Revision", "[subject name]"],
     description: "Spaced repetition review due. Last studied: [date]. Review type: [quick/practice/teach-back]."
   )
   ```

4. **For integration project opportunities:**
   ```
   DailyPlanner-create_task(
     title: "Project: [project name]",
     type: "Task",
     priority: "P3",
     tags: ["Learning", "Project", "[subject A]", "[subject B]"],
     description: "Integration project combining [topics]. Build: [project description]."
   )
   ```

5. **Summary of actions taken:**
   ```markdown
   ## ✅ Actions Created
   | Type | Count | Details |
   |------|-------|---------|
   | Study tasks | [N] | [topic list] |
   | Revision tasks | [M] | [topic list] |
   | Learning paths | [K] | [domain list] |
   | Projects | [P] | [project list] |
   ```

## Tools & APIs Used
- `DailyPlanner-get_subjects` — All subjects
- `DailyPlanner-get_subject` — Subject detail
- `DailyPlanner-get_topics` — All topics with progress
- `DailyPlanner-get_resources` — Resources and completion
- `DailyPlanner-get_learning_focus` — Priority suggestions
- `DailyPlanner-get_goals` — Career goal alignment
- `DailyPlanner-create_task` — Create study/revision/project tasks from gaps
- `notion-API-post-search` — Check notes existence
- `notion-API-get-block-children` — Notes quality check

## Output Format
Multi-section learning dashboard: overview metrics → subject progress (grouped by status) → gap analysis → career alignment → recommendations → spaced review.

## Notes
- Staleness (days since last activity) is a key indicator — flag anything >30 days
- Career-aligned subjects should always be prioritized in recommendations
- Topics with high progress but no notes suggest retention risk
- Encourage archiving subjects that are no longer relevant to reduce noise
- Spaced review intervals: 1, 3, 7, 14, 30, 90 days after completion
- This skill pairs well with `periodic-review` for holistic life review

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
  description = "Surfaced by: learning-review · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "learning-review"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.