---
description: >
  Analyze a Copilot session to generate a better title and detect when a session
  covers multiple unrelated topics that should be split into distinct tracked items.
  Use this skill when the user says 'rename session', 'fix session title',
  'session title', 'better session name', 'what was this session about',
  'split session', 'session has too many topics', or 'organize session'.
  Generates descriptive titles, and when multiple unrelated work streams are detected,
  creates distinct Notion pages and Daily Planner tasks with full context and
  a reference session ID so follow-up sessions can query back for history.
---

# Session Rename — Better Titles & Multi-Topic Detection

## Context
Auto-generated session titles are often vague ("Copy Local Skills To Repo") or capture only the first topic discussed. Real sessions frequently drift across unrelated topics — debugging a pipeline, then drafting an email, then designing a feature. This skill analyzes session content, generates a precise title, and when multiple unrelated work streams are detected, splits them into distinct tracked items in Notion and Daily Planner so the user can continue each in a focused follow-up session.

## Why This Matters
- **Findability:** Sessions with good titles are searchable. "Copy Local Skills To Repo" vs. "Skills Marketplace: Create sync-to-github and sync-from-github skills" — one is useful in 6 months, the other isn't.
- **Context continuity:** When a session covers 3 topics, a single title loses 2 of them. Splitting into tracked items preserves all context.
- **Follow-up sessions:** By storing the reference session ID in Notion/Daily Planner, a new session can query the session store for full history, picking up exactly where the user left off.

## When to Use
- When a session title feels wrong or too vague
- At the end of a long session that covered multiple topics
- When invoked by `close-day` or `session-outcomes` for sessions that need better titles
- When the user wants to split a multi-topic session into trackable work items

---

## Instructions

### Step 1: Identify the Target Session

#### If the user specifies a session:
Use the provided session ID or search by keyword:
```sql
-- session_store_sql (scope: personal)
SELECT id, summary, repository, branch, created_at, updated_at
FROM sessions
WHERE summary ILIKE '%{user_input}%' OR id = '{user_input}'
ORDER BY updated_at DESC
LIMIT 10
```

#### If no session is specified:
Default to the **current session**. Use the current conversation context directly — no need to query the session store.

#### If invoked by another skill (close-day, session-outcomes):
Use the session ID passed by the calling skill.

### Step 2: Gather Session Content

Pull all available context for the session:

```sql
-- session_store_sql (scope: personal)
-- Checkpoints (richest summary source)
SELECT checkpoint_number, title, overview
FROM checkpoints
WHERE session_id = '{session_id}'
ORDER BY checkpoint_number
```

```sql
-- session_store_sql (scope: personal)
-- User messages (understand intent and topic shifts)
SELECT turn_index, substr(COALESCE(user_message, ''), 1, 500) as user_message
FROM turns
WHERE session_id = '{session_id}'
ORDER BY turn_index
```

```sql
-- session_store_sql (scope: personal)
-- Files touched (indicate what was actually worked on)
SELECT file_path, tool_name, turn_index
FROM session_files
WHERE session_id = '{session_id}'
ORDER BY turn_index
```

```sql
-- session_store_sql (scope: personal)
-- Refs (commits, PRs, issues linked)
SELECT ref_type, ref_value, turn_index
FROM session_refs
WHERE session_id = '{session_id}'
ORDER BY turn_index
```

### Step 3: Analyze Topics

Examine the gathered content to identify **distinct work streams**. A work stream is a logically cohesive unit of work — it has a clear goal, related files, and a beginning/end within the session.

#### Topic Detection Signals:
- **User intent shifts:** The user changes what they're asking about (e.g., from "fix the build" to "draft an email to Duncan")
- **File path divergence:** Work switches from one repo/directory to a completely different one
- **Tool usage shifts:** From code editing to email drafting, from task management to calendar operations
- **Checkpoint boundaries:** Checkpoints often mark natural topic boundaries
- **Temporal gaps:** Long pauses between turns may indicate topic switches

#### For each detected topic, extract:
```
{
  "topic_number": 1,
  "title": "Descriptive title for this work stream",
  "description": "2-3 sentence summary of what was done and the current state",
  "turns": [start_turn, end_turn],
  "files_touched": ["list of relevant files"],
  "refs": ["commits, PRs, issues"],
  "status": "completed | in-progress | abandoned",
  "next_steps": "What would the user do next to continue this work",
  "tags": ["relevant tags for Daily Planner — e.g., 'official', 'personal', team tags"]
}
```

### Step 4: Generate Better Session Title

Based on the topic analysis:

#### Single-topic session:
Generate a descriptive title that captures:
- **What** was done (verb + noun): "Create", "Fix", "Design", "Debug", "Draft"
- **Where** it was done (repo, service, feature area)
- **Key outcome** if completed

**Title format:** `{Action}: {Specific subject} [{repo/context}]`

**Good examples:**
- `Skills Marketplace: Create sync-to-github and sync-from-github skills`
- `Trade Management: Design backtest feeder architecture with event-driven pipeline`
- `Daily Planner API: Fix meeting sync duplication bug`
- `Duncan 1:1 Prep: Q2 goals review and blocker escalation`

**Bad examples (avoid):**
- `Working on stuff` — no specifics
- `Bug fix` — which bug? where?
- `Code changes` — meaningless
- `Session work` — circular

#### Multi-topic session:
Generate a title that acknowledges the breadth:
- `Multi-topic: {primary topic} + {N} other items`
- Or name the top 2-3 topics: `Skills sync setup, backtest architecture, Duncan prep`

### Step 5: Present Analysis to User

```markdown
## 📝 Session Analysis

**Current title:** {current_summary}
**Suggested title:** {new_title}

### Topics Detected: {N}

| # | Topic | Turns | Status | Files |
|---|-------|-------|--------|-------|
| 1 | {title} | {start}-{end} | ✅ Completed | {count} files |
| 2 | {title} | {start}-{end} | 🔄 In Progress | {count} files |
| 3 | {title} | {start}-{end} | ⏸️ Abandoned | {count} files |
```

If **single topic** — offer to rename:
```
ask_user:
  question: "Should I rename this session?"
  choices:
    - "Yes, use the suggested title (Recommended)"
    - "Let me provide a custom title"
    - "Keep the current title"
```

If **multiple topics** — offer to split:
```
ask_user:
  question: "This session covers {N} unrelated topics. Should I create separate tracked items for each?"
  choices:
    - "Yes, create Notion pages + Daily Planner tasks for each (Recommended)"
    - "Just rename the session, don't split"
    - "Let me review the topics first"
```

### Step 6: Rename the Session (if approved)

The session store's `summary` field is the title. Update it:

> **Note:** The session store is read-only from the `session_store_sql` tool. To rename, use the `store_memory` tool to record the preferred title, and inform the user that the session title in the UI will update on the next checkpoint or can be manually edited.

If direct rename is not possible, log the preferred title:
```
DailyPlanner-add_activity_log(
  taskId: "{related_task_id}",
  description: "Session {session_id} renamed: '{new_title}' (was: '{old_title}')"
)
```

### Step 7: Create Distinct Items for Multi-Topic Sessions

For each detected topic (when the user approves splitting):

#### 7a: Create Notion Page

Create a page under the appropriate parent (personal workspace or team page):

```
notion-mcp-create_page(
  parentId: "{appropriate_parent_page_id}",
  parentType: "page",
  title: "📋 {topic_title}"
)
```

Add structured content to the page:

```
notion-mcp-append_block_children(
  blockId: "{new_page_id}",
  childrenJson: [
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": { "rich_text": [{ "type": "text", "text": { "content": "Context" } }] }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": { "rich_text": [{ "type": "text", "text": { "content": "{topic_description}" } }] }
    },
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": { "rich_text": [{ "type": "text", "text": { "content": "Reference Session" } }] }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": { "rich_text": [{ "type": "text", "text": { "content": "Session ID: {session_id}\nSession Title: {session_title}\nDate: {session_date}\nTurns: {start_turn} to {end_turn}\nRepository: {repository}" } }] }
    },
    {
      "object": "block",
      "type": "callout",
      "callout": {
        "icon": { "type": "emoji", "emoji": "💡" },
        "rich_text": [{ "type": "text", "text": { "content": "To continue this work, start a new session and ask:\n\"Pull context for session {session_id} about {topic_title}\"" } }]
      }
    },
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": { "rich_text": [{ "type": "text", "text": { "content": "Work Done So Far" } }] }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": { "rich_text": [{ "type": "text", "text": { "content": "{detailed summary of what was accomplished}" } }] }
    },
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": { "rich_text": [{ "type": "text", "text": { "content": "Files Touched" } }] }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": { "rich_text": [{ "type": "text", "text": { "content": "{list of files with what was changed}" } }] }
    },
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": { "rich_text": [{ "type": "text", "text": { "content": "Next Steps" } }] }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": { "rich_text": [{ "type": "text", "text": { "content": "{what needs to happen next to complete this work}" } }] }
    }
  ]
)
```

#### 7b: Create Daily Planner Task

For each topic that is **in-progress** or has clear **next steps**:

```
DailyPlanner-create_task(
  title: "{topic_title}",
  description: "Extracted from session {session_id} on {date}.\n\n{topic_description}\n\nNext steps: {next_steps}\n\nNotion page: {notion_page_url}\nReference session: {session_id}\n\nTo continue, start a new Copilot session and query:\nsession_store_sql: SELECT * FROM turns WHERE session_id = '{session_id}' AND turn_index BETWEEN {start_turn} AND {end_turn}",
  priority: "{P2 for official work, P3 for personal}",
  tags: "{comma-separated tags}",
  type: "Task"
)
```

#### 7c: Link Task to Notion Page

After creating both, add the task ID to the Notion page:

```
notion-mcp-append_block_children(
  blockId: "{notion_page_id}",
  childrenJson: [
    {
      "object": "block",
      "type": "heading_2",
      "heading_2": { "rich_text": [{ "type": "text", "text": { "content": "Daily Planner Task" } }] }
    },
    {
      "object": "block",
      "type": "paragraph",
      "paragraph": { "rich_text": [{ "type": "text", "text": { "content": "Task ID: {task_id}\nTitle: {task_title}\nPriority: {priority}\nStatus: New" } }] }
    }
  ]
)
```

### Step 8: Report Results

```markdown
## ✅ Session Organized

### Title Updated
**Before:** {old_title}
**After:** {new_title}

### Items Created ({N} topics split)

| # | Topic | Notion Page | Daily Planner Task | Status |
|---|-------|-------------|-------------------|--------|
| 1 | {title} | ✅ [Page]({url}) | ✅ {task_id} | In Progress |
| 2 | {title} | ✅ [Page]({url}) | ✅ {task_id} | New |
| 3 | {title} | ✅ [Page]({url}) | — (completed) | Done |

### How to Continue Each Topic
| Topic | Command |
|-------|---------|
| {title_1} | `start task {task_id}` |
| {title_2} | `start task {task_id}` |

### Session Reference
Each Notion page and task includes:
- 🔗 **Session ID:** `{session_id}` — for querying full session history
- 📍 **Turn range:** Which turns in the session relate to this topic
- 📂 **Files:** Which files were touched for this topic
- ➡️ **Next steps:** What to do next

> 💡 In a new session, the task description contains the exact `session_store_sql`
> query to pull context from the original session.
```

---

## Integration with Other Skills

### session-outcomes integration
When `session-outcomes` processes sessions, it can invoke this skill for sessions with vague titles:
- If `summary` is NULL or less than 20 characters → auto-invoke session-rename
- If multiple distinct outcomes are detected → suggest splitting

### close-day integration
During `close-day`, after extracting outcomes:
- Flag sessions with poor titles for renaming
- Flag sessions with 3+ topics for splitting

### session-knowledge integration
When splitting a multi-topic session, also invoke `session-knowledge` for each topic:
- Extract reusable knowledge per topic
- Route knowledge to the right destination (instructions, skills, Notion)

### start-task integration
When starting a task that was created by this skill:
- The task description contains the session ID and turn range
- Query `session_store_sql` to pull the relevant conversation history
- Present the context to the user as a starting point

---

## Context Retrieval for Follow-Up Sessions

When a user starts a new session to continue work from a split topic, the task description includes a ready-to-use query:

```sql
-- Pull all relevant turns from the original session
SELECT turn_index, user_message, substr(COALESCE(assistant_response, ''), 1, 2000) as assistant_response
FROM turns
WHERE session_id = '{original_session_id}'
  AND turn_index BETWEEN {start_turn} AND {end_turn}
ORDER BY turn_index
```

```sql
-- Pull files that were modified during this topic
SELECT file_path, tool_name, turn_index
FROM session_files
WHERE session_id = '{original_session_id}'
  AND turn_index BETWEEN {start_turn} AND {end_turn}
ORDER BY turn_index
```

This gives the new session full context without the user having to re-explain anything.

---

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Session has only 1-2 turns | Generate a title from the user's first message; skip topic detection |
| Session is current (still active) | Work with conversation context directly; note that more topics may emerge |
| All topics are closely related | Don't split — treat as a single coherent session with a good compound title |
| No checkpoints available | Rely on turns and file changes for analysis |
| User disagrees with topic boundaries | Let them adjust — present the analysis as a suggestion, not a final decision |
| Session belongs to a different repo | Include the repo name in the title and tag tasks appropriately |
| Topic has no files touched | Still valid (e.g., email drafting, meeting prep) — track by tool usage instead |

---

## Tools & APIs Used

### Session Store (read-only)
- `session_store_sql` — Query sessions, turns, checkpoints, files, refs

### Notion
- `notion-mcp-create_page` — Create topic pages
- `notion-mcp-append_block_children` — Add structured content
- `notion-mcp-search` — Find existing pages to avoid duplicates

### Daily Planner
- `DailyPlanner-create_task` — Create tasks for in-progress topics
- `DailyPlanner-add_activity_log` — Log renaming activity
- `DailyPlanner-search_tasks` — Check for existing tasks to avoid duplicates

### User Interaction
- `ask_user` — Confirm title, splitting decisions

### Other Skills
- `session-knowledge` — Extract knowledge per topic when splitting
- `session-outcomes` — Caller skill, provides session list

---

## Output Format
- Single-topic: Suggested title with confirmation prompt
- Multi-topic: Analysis table + Notion pages + Daily Planner tasks + continuation instructions

## Notes
- The session store is read-only — title updates are tracked via Daily Planner activity logs
- Always include the session ID in Notion pages and task descriptions for traceability
- Turn ranges are inclusive — `BETWEEN start AND end` captures the full topic
- When in doubt about topic boundaries, ask the user — they know their intent best
- Completed topics still get Notion pages (for documentation) but not Daily Planner tasks
