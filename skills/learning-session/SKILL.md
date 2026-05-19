---
description: "Run a focused learning session with Notion notes and Daily Planner progress tracking. Use this skill when the user says 'start learning', 'study session', 'learn', 'study [topic]', 'learning time', 'what should I study', or 'continue studying'. Provides resource guidance, captures notes to Notion, and tracks progress in Daily Planner."
---

# Learning Session Skill

## Context
Learning is tracked across two systems:
- **Daily Planner** — Progress tracking (subjects, topics, resources, percentages, time spent)
- **Notion** — Study notes, key concepts, code snippets, questions (rich content)

This skill runs a complete study session: picks what to study, provides the resource, captures notes during the session, and updates progress when done.

## When to Use
- When the user wants to study or learn
- When looking for what to study next
- When continuing a previous study session
- When tracking learning progress

## Workflow

### Step 1: Get Learning Suggestions
```
DailyPlanner-get_learning_focus(limit: 5)
```

Returns in-progress topics sorted by least progress. Also check:
```
DailyPlanner-get_subjects(status: "Active")
```

### Step 2: Choose a Topic
Present suggestions with context:
```markdown
## 📚 What to Study

| # | Subject | Topic | Progress | Next Block | Block Duration | Priority |
|---|---------|-------|----------|-----------|---------------|----------|
| 1 | Observability | Logs Pillar | 10% | Block 2: Quickstart tutorial | 45 min | 🔴 Least progress |
| 2 | .NET Aspire | Getting Started | 25% | Block 3: Architecture patterns | 30 min | 🟡 In progress |
| 3 | Kubernetes | Services | 40% | Block 4: Mini-project pt 1 | 45 min | 🟢 Active |
```

```
ask_user: "What would you like to study? How much time do you have?"
  choices: ["[Topic 1] — 45 min block", "[Topic 2] — 30 min block", "[Topic 3] — 45 min block", "Something else"]
```

**Time-aware selection:** If the user specifies available time (e.g., "I have 30 minutes"), filter to blocks that fit within that window.

### Step 3: Load Session Context
Once a topic is selected, pull everything:

1. **Topic resources:**
   ```
   DailyPlanner-get_resources(topicId: "[topic_id]")
   ```

2. **Previous notes from Notion:**
   Search for the topic's Notion page:
   ```
   notion-API-post-search(query: "[topic name]", filter: { property: "object", value: "page" })
   ```
   If found, load recent notes:
   ```
   notion-API-get-block-children(block_id: "[topic_page_id]")
   ```

3. **Subject context:**
   ```
   DailyPlanner-get_subject(subjectId: "[subject_id]")
   ```

#### Load Hands-On Materials
For the selected topic, prepare practical materials:

1. **Find relevant GitHub repos:**
   ```
   github-mcp-server-search_repositories: "[topic] tutorial OR example OR starter" 
   ```
   Select 2-3 repos with good documentation to reference during the session.

2. **Find current blog posts/articles:**
   ```
   web_search: "[topic] practical guide [current year]"
   web_search: "[topic] best practices for architects"
   ```
   Provide 2-3 relevant articles to read or reference.

3. **Prepare exercise:**
   Based on the topic, prepare a hands-on exercise:
   - **Beginner:** Follow along with official tutorial or quickstart
   - **Intermediate:** Modify an existing example to add a feature
   - **Advanced:** Build something from scratch or solve an architecture problem

4. **Load cross-topic connections:**
   Check what other topics connect to this one:
   ```
   DailyPlanner-get_topics(subjectId: "[subject_id]")
   ```
   Identify related topics and suggest: "This connects to [topic X] — consider exploring that next."

### Step 4: Present Session Setup

```markdown
# 📖 Learning Session: [Topic Name]
**Subject:** [Subject] | **Progress:** [X]% | **Session #:** [N]

## 📚 Resources
| # | Resource | Type | Progress | URL |
|---|----------|------|----------|-----|
| 1 | [Title] | Video | ▓▓▓░░ 60% | [link] |
| 2 | [Title] | Article | ░░░░░ 0% | [link] |

## 📝 Previous Notes Summary
[Last session's key takeaways from Notion, or "No previous notes — this is your first session on this topic"]

## ❓ Open Questions from Last Session
[Questions noted in Notion's "Questions & Gaps" section]

## 🎯 Session Goal
Continue with: **[Resource title]** ([type]) — pick up from [X]%
```

Choose which resource to work on:
```
ask_user: "Which resource do you want to work on?"
  choices: ["[Resource 1] (continue from 60%)", "[Resource 2] (start fresh)"]
```

#### Session Resources Panel
Present resources for this session:
```markdown
## 📚 Session Resources: [Topic]

### 🔧 Hands-On Exercise
[Description of the practical exercise for this session]
**Difficulty:** [Beginner/Intermediate/Advanced]
**Estimated time:** [X min]

### 💻 Reference Repos
| Repo | Why Study It | Key Files |
|------|-------------|-----------|
| [owner/repo] | [what to learn from it] | [files to focus on] |

### 📖 Reading Material
- [Blog/article title] — [link] — [why it's relevant]
- [Book chapter] — [reference]

### 🔗 Connections to Other Topics
- This topic is a prerequisite for: [topic A], [topic B]
- Related to: [topic C] in [subject Y]
- After this, consider: [next logical topic]
```

### Step 5: During the Session
The session is active. As the user studies, they may:

**Take notes** (captured to Notion):
When the user shares notes, key points, or code snippets during the session, append to the topic's Notion page under the appropriate section:

```
notion-API-patch-block-children(block_id: "[topic_page_id]", children: [
  { "heading_3": { "rich_text": [{ "text": { "content": "Session — [Date]" } }] } },
  { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[note from user]" } }] } }
])
```

**Key concepts** → Append under "💡 Key Concepts" section
**Code snippets** → Append under "💻 Code Snippets" section as code blocks:
```json
{ "code": { "rich_text": [{ "text": { "content": "[code]" } }], "language": "[language]" } }
```
**Questions** → Append under "❓ Questions & Gaps" section

**Ask for explanations:**
The user may ask for clarification on concepts. Use `web_search` or existing knowledge to explain.

**Summarize content:**
If studying a video or article, offer to summarize key sections:
```
web_fetch(url: "[resource URL]") — for articles
web_search(query: "[topic] [specific concept]") — for deeper understanding
```

#### Capture Practical Learnings
During the session, capture not just concepts but practical artifacts:

1. **Code snippets** — Working examples you wrote or found
2. **Architecture patterns** — Diagrams or patterns discovered (Mermaid)
3. **Commands/procedures** — Runbook-style steps for common operations
4. **Trade-offs learned** — When to use vs. not use this technology
5. **"Aha moments"** — Key insights that changed your understanding
6. **Questions for deeper study** — Things you want to explore further

### Step 6: End Session
When the user says they're done, or when the current learning block's time is up:

1. **Verify Definition of Done:**
   Check the current block's DoD criteria:
   ```
   📋 Block DoD Check: "[block title]"
   - [ ] [DoD criterion 1]
   - [ ] [DoD criterion 2]
   ```
   Ask the user: "Did you complete the Definition of Done for this block?"
   - If yes: mark block complete, update progress
   - If partially: note what's left, mark for continuation next session
   - If no: keep block in progress, note where to resume

2. **Update progress:**
   ```
   ask_user: "How far did you get with this resource? (percentage)"
   ```

**Update Daily Planner progress:**
```
DailyPlanner-update_learning_progress(
  targetType: "resource",
  targetId: "[resource_id]",
  progress: [new_percentage],
  timeSpent: [minutes_spent]
)
```

Calculate and update topic progress (average of resource progress):
```
DailyPlanner-update_learning_progress(
  targetType: "topic",
  targetId: "[topic_id]",
  progress: [calculated_average]
)
```

**Save session summary to Notion:**
Append a session summary block to the topic page:
```
notion-API-patch-block-children(block_id: "[topic_page_id]", children: [
  { "divider": {} },
  { "callout": {
    "rich_text": [{ "text": { "content": "📊 Session [Date] — [X] min — Progress: [old]% → [new]%" } }],
    "icon": { "emoji": "📊" }
  }},
  { "heading_3": { "rich_text": [{ "text": { "content": "Key Takeaways" } }] } },
  { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[takeaway 1]" } }] } },
  { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[takeaway 2]" } }] } },
  { "heading_3": { "rich_text": [{ "text": { "content": "Next Session" } }] } },
  { "paragraph": { "rich_text": [{ "text": { "content": "Continue from: [where to pick up]" } }] } }
])
```

### Step 7: Session Summary

```markdown
## ✅ Learning Session Complete

| Detail | Value |
|--------|-------|
| **Subject** | [Subject name] |
| **Topic** | [Topic name] |
| **Resource** | [Resource title] |
| **Time spent** | [X] minutes |
| **Resource progress** | [old]% → [new]% |
| **Topic progress** | [X]% |
| **Notes saved** | ✅ Notion |
| **Progress updated** | ✅ Daily Planner |

### 📝 Session Notes Captured
- [X] key concepts
- [X] code snippets
- [X] questions to explore

### 🎯 Next Session Suggestion
**Topic:** [next topic or continue current]
**Resource:** [next resource]
**Why:** [least progress / sequence order / builds on today]
```

#### Cross-Topic Suggestions
At the end of each session, suggest connections:

1. **Related topics to explore next:**
   Based on what was studied, suggest 2-3 related topics across subjects.

2. **Integration project ideas:**
   If this topic + previously learned topics could form a project, suggest it:
   ```
   💡 Project idea: You've now covered [topic A] and [topic B]. 
   Consider building: [project description that combines both]
   ```

3. **Revision reminder:**
   Schedule the next review based on spaced repetition:
   - If first time: review in 3 days
   - If second review: review in 7 days
   - If third review: review in 14 days
   ```
   📅 Next review of [topic] scheduled for [date] — I'll remind you during `start-day`.
   ```

#### Update Existing Related Notes
After each session, check if what was learned enriches existing knowledge:

1. **Search related Notion pages:**
   ```
   notion-API-post-search(query: "[key concepts learned today]")
   ```
   Look for other topic pages that reference or relate to today's learning.

2. **Cross-reference updates:**
   For each related topic page found:
   - Add a "Related: [today's topic]" link in the Connections section
   - If today's learning clarifies or expands a concept in another topic, append a note:
     ```
     notion-API-patch-block-children: "📎 Updated from [topic] session on [date]: [insight that enriches this topic]"
     ```

3. **Update subject-level notes:**
   If the session revealed connections or insights at the subject level:
   - Update the subject's Notion page with new cross-topic connections
   - Add to the "Key Insights" section if the subject page has one

4. **Notify about stale notes:**
   If related topic notes haven't been reviewed in 30+ days and today's session revealed updates:
   > "📝 Your notes on [related topic] may need updating based on today's learning. Schedule a review?"

## Tools & APIs Used
- `DailyPlanner-get_learning_focus` — Topic suggestions (by least progress)
- `DailyPlanner-get_subjects` / `get_subject` — Subject details
- `DailyPlanner-get_topics` — Topic listing
- `DailyPlanner-get_resources` — Topic resources
- `DailyPlanner-update_learning_progress` — Progress tracking (resource + topic)
- `notion-API-post-search` — Find topic's Notion page
- `notion-API-get-block-children` — Load previous notes
- `notion-API-patch-block-children` — Save notes, summaries, code snippets
- `web_search` / `web_fetch` — Research and explain concepts
- `ask_user` — Topic selection, progress input, note capture

## Output Format
Session setup with context → active note-taking → progress update → session summary with next steps.

## Notes
- Always load previous notes at session start — continuity matters
- Capture notes in real-time to Notion as the user shares them
- Code snippets should include the language tag for proper formatting
- Questions logged during the session become the starting point for next session
- Time tracking helps estimate total learning investment per subject
- If no Notion page exists for a topic, suggest running `learning-setup` first

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
  description = "Surfaced by: learning-session · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "learning-session"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.