---
name: session-outcomes
description: >
  Extract distinct work outcomes from Copilot sessions, classify as personal or
  official, and track in Daily Planner. Use this skill when the user says
  "review my sessions", "extract session outcomes", "what did I do today",
  "session work", "untracked work", or "process sessions". Also invoked by
  close-day (today's sessions) and start-day (yesterday's catch-up).
---

# Session Outcomes — Extract & Track Work from Copilot Sessions

Analyzes Copilot sessions to find distinct pieces of work, classifies them as personal or official, and creates/links Daily Planner tasks.

## Why This Skill Exists

Long sessions accumulate multiple distinct work items that go untracked. This skill ensures every piece of work — whether a code change, email draft, research spike, or meeting prep — is captured, classified, and tracked.

## Trigger Modes

| Mode | Trigger | Scope |
|------|---------|-------|
| **Ad-hoc** | "review my sessions", "extract session outcomes", "what did I do today" | User-specified date range or session |
| **Integrated (close-day)** | Called by `close-day` skill | Today's sessions (created_at or updated_at = today) |
| **Integrated (start-day)** | Called by `start-day` skill | Yesterday's sessions (catch unprocessed work) |
| **Targeted** | "review session {id or name}" | Specific session by ID or summary match |

---

## Instructions

### Step 0: Determine Scope

Based on how the skill was triggered:

#### Ad-hoc invocation:
Ask the user:
```
ask_user:
  question: "What sessions should I analyze?"
  choices:
    - "Today's sessions (Recommended)"
    - "Yesterday's sessions"
    - "This week's sessions"
    - "A specific session"
```

If "specific session": ask for session ID or name, then search:
```sql
SELECT id, summary, created_at, updated_at
FROM sessions
WHERE summary LIKE '%{user_input}%' OR id = '{user_input}'
ORDER BY created_at DESC LIMIT 10
```

#### From close-day:
Auto-scope to today:
```sql
SELECT s.*, COUNT(t.id) as turn_count
FROM sessions s
JOIN turns t ON s.id = t.session_id
WHERE date(s.updated_at) = date('now')
GROUP BY s.id
ORDER BY s.created_at
```

#### From start-day:
Auto-scope to yesterday:
```sql
SELECT s.*, COUNT(t.id) as turn_count
FROM sessions s
JOIN turns t ON s.id = t.session_id
WHERE date(s.updated_at) = date('now', '-1 day')
GROUP BY s.id
ORDER BY s.created_at
```

**Skip sessions with < 3 turns** — these are trivial and unlikely to contain meaningful work.

### Step 0.1: Verify Daily Planner Tools Are Available

**CRITICAL: Do NOT proceed without Daily Planner tools.** All outcomes must be tracked there.

1. **Check if `DailyPlanner-*` tools are available** in the current session by attempting:
   ```
   DailyPlanner-search_tasks(query: "test")
   ```

2. **If tools are NOT available**, work with the user to fix it:
   ```
   ask_user:
     question: "Daily Planner MCP tools are not available in this session. We need them to track outcomes. How should we proceed?"
     choices:
       - "Restart the session (I'll re-invoke after restart)"
       - "Kill and restart the Daily Planner MCP server"
       - "Skip outcome tracking for now"
   ```

   **To restart the Daily Planner MCP:**
   - Find and kill existing instances: `Get-Process -Name "node" | Where-Object { $_.CommandLine -like "*daily-planner*" } | Stop-Process`
   - Or check MCP server config in VS Code settings / Copilot config
   - The user may need to restart their IDE or Copilot CLI session

3. **If tools ARE available**, proceed to Step 0.5.

**Do NOT fall back to Notion or any other system.** Daily Planner is the single source of truth for task tracking.

### Step 0.5: Sync Sessions to Backend

Before analyzing sessions, sync them to the Daily Planner backend so they're accessible from the web UI:

```
DailyPlanner-sync_copilot_sessions(since: "{scope_start_date}")
```

This ensures the sessions being analyzed are also viewable in the Daily Planner web app at kamin.day. Report the sync count briefly and continue.

### Step 1: Gather Session Data

For each session in scope, collect data in parallel:

1. **Checkpoints first** (most structured, highest signal):
   ```sql
   SELECT checkpoint_number, title, overview, work_done, technical_details, important_files, next_steps
   FROM checkpoints
   WHERE session_id = '{session_id}'
   ORDER BY checkpoint_number
   ```

2. **User messages** (understand intent and topic shifts):
   ```sql
   SELECT turn_index, user_message, timestamp
   FROM turns
   WHERE session_id = '{session_id}'
   ORDER BY turn_index
   ```

3. **Files modified** (what was produced):
   ```sql
   SELECT file_path, tool_name, turn_index
   FROM session_files
   WHERE session_id = '{session_id}'
   ORDER BY turn_index
   ```

4. **Refs** (commits, PRs, issues — concrete deliverables):
   ```sql
   SELECT ref_type, ref_value, turn_index
   FROM session_refs
   WHERE session_id = '{session_id}'
   ORDER BY turn_index
   ```

5. **Session metadata**:
   ```sql
   SELECT id, cwd, repository, branch, summary, created_at, updated_at
   FROM sessions
   WHERE id = '{session_id}'
   ```

**Efficiency rule:** If checkpoints exist and provide sufficient detail, skip deep turn-by-turn analysis. Only drill into turns when:
- No checkpoints exist
- Checkpoints don't cover all the work (turn count >> checkpoint coverage)
- Need to pinpoint topic transitions

### Step 2: Extract Distinct Work Items

Analyze the gathered data to identify **distinct pieces of work**. A distinct work item is a logically separate effort with its own goal/outcome.

#### Detection Signals

**Explicit topic changes (highest confidence):**
- User says: "now let's work on...", "next I want to...", "switching to...", "let's move on to..."
- User references a different task ID or project
- User starts a completely new topic after completing previous work

**Context shifts (high confidence):**
- Different repository or branch appears in files/refs
- Checkpoint boundary with a different title/topic
- CWD changes to a different project

**Implicit boundaries (medium confidence):**
- Significant time gap between turns (> 30 minutes)
- Domain shift: code work → email drafting → documentation → meeting prep
- Different tool usage patterns: coding tools → communication tools → planning tools

#### For Each Distinct Work Item, Extract:

```
{
  "title": "Concise descriptive title",
  "description": "What was done and why (2-3 sentences)",
  "type": "Feature | BugFix | Research | Documentation | Communication | Admin | Learning | MeetingPrep | Infrastructure | Review",
  "repository": "repo name or null",
  "branch": "branch name or null",
  "files": ["key files created/modified"],
  "refs": {
    "commits": ["sha1", "sha2"],
    "prs": ["#123"],
    "issues": ["#456"]
  },
  "outcome": "What was produced/achieved",
  "turnRange": [start_turn, end_turn],
  "timestamps": {
    "start": "ISO timestamp",
    "end": "ISO timestamp"
  },
  "durationEstimate": "~30 minutes",
  "classification": "official | personal | ambiguous",
  "classificationReason": "Why it was classified this way",
  "relatedTeam": "Team name or null",
  "existingTaskId": "Daily Planner task ID if matched, or null"
}
```

### Step 3: Classify Each Work Item (Personal vs Official)

Apply these rules in order. First match wins.

#### Official Signals (any match → **official**):

1. **Repository match:** Repository belongs to a known team:
   | Repository Pattern | Team |
   |---|---|
   | `reliability.tools.ooa`, `reliability.tools.*` | Reliability Data Engineering |
   | `bangtest*`, `bang-*` | Reliability Data Engineering |
   | `rde-*` | Reliability Data Engineering |
   | `benchmarking*`, `fungates*` | Benchmarking |
   | `anomaly*`, `alerting*` | Anomaly Detection |
   | `sustainability*` | Sustainability |
   | `performance*`, `power-*` | Performance & Sustainability DE |
   | `gates*`, `defense*` | Gates & Defense |
   | `cosine*` | COSINE |
   | `duma*` | Team Duma |

2. **Existing task tagged official:** If the work matches a Daily Planner task with an "official" or "Official" tag

3. **Content signals:** Work references Microsoft internal tools, teams, ADO boards, internal URLs, team members by name

4. **Meeting correlation:** Work was discussed in or resulted from a work calendar meeting

5. **CWD match:** Session working directory is in an official repo workspace

#### Personal Signals (→ **personal**):

1. **Personal repos:** `copilot-session-viewer`, `personal-copilot-agent`, `learning`, personal GitHub repos
2. **Personal domains:** Health/fitness tracking, financial management, personal learning not related to work goals, journal entries
3. **Personal project patterns:** Side projects, hobby code, personal automation

#### Ambiguous (→ **ask user**):

If no clear signal matches, present the work item and ask:
```
ask_user:
  question: "I couldn't confidently classify this work item:\n\n**{title}**\n{description}\n\nIs this official (Microsoft/team work) or personal?"
  choices: ["Official — {suggested_team}", "Personal", "Skip — don't track this"]
```

### Step 4: Match Existing Daily Planner Tasks

For each work item, check if a task already exists:

1. **Session name check:** If session summary contains a task ID pattern (e.g., "142-Build Email Template"), extract and match:
   ```
   DailyPlanner-get_task(taskId: "{extracted_id}")
   ```

2. **Ref check:** If session_refs contains issue/PR references, search for linked tasks:
   ```
   DailyPlanner-search_tasks(query: "#{ref_value}")
   ```

3. **Keyword search:** Search by work item title:
   ```
   DailyPlanner-search_tasks(query: "{work_item_title_keywords}")
   ```

4. **Tag search:** If team is identified, search by team tag:
   ```
   DailyPlanner-get_tasks(tag: "{team_tag}")
   ```

#### If Match Found:
- Link the outcome to the existing task
- Update with activity log:
  ```
  DailyPlanner-add_activity_log(
    taskId: "{task_id}",
    description: "📋 SESSION OUTCOME: {title}\n\n{description}\n\nDeliverables: {outcome}\nSession: {session_id}",
    durationMinutes: {estimated_duration}
  )
  ```
- If the work completed the task:
  ```
  DailyPlanner-complete_task(
    taskId: "{task_id}",
    summary: "{outcome summary}"
  )
  ```

#### If No Match Found — Create New Task:
```
DailyPlanner-create_task(
  title: "{work_item_title}",
  description: "{work_item_description}\n\nExtracted from session: {session_id}\nSession: {session_summary}",
  priority: "{inferred_priority}",
  tags: "{classification_tag}, {team_tag}, session-extracted",
  type: "{inferred_type}"
)
```

**Tag rules:**
- Official work: Add "official" + team tag (e.g., "official, Reliability Data Engineering, session-extracted")
- Personal work: Add "personal" + category tag (e.g., "personal, side-project, session-extracted")
- Always add "session-extracted" tag for tracking

**Priority inference:**
- Work with commits/PRs → P2 (substantial, already in progress)
- Email/communication sent → P3 (completed admin)
- Research/documentation → P3 (standard priority)
- Meeting prep → P3 (standard)
- If uncertain → P3 (default)

### Step 5: Enrich with External Context

For each work item, pull related context from external sources. Run these in parallel:

1. **WorkIQ — Related communications:**
   ```
   workiq-ask_work_iq:
     question: "What emails, Teams messages, or meetings from {date} relate to '{work_item_title}'? Summarize who was involved and key points."
   ```

2. **Calendar — Meeting correlation:**
   ```
   DailyPlanner-get_todays_meetings(date: "{session_date}")
   ```
   Check if any meetings overlap with the session timeframe and relate to the work topic.

3. **Teams — Related discussions (only if work item is official):**
   ```
   teams-SearchTeamMessagesQueryParameters:
     queryString: "{key_terms_from_work_item}"
     size: 5
   ```

Add enrichment data to the work item:
- Related people (who else was involved)
- Meeting context (was this work discussed/assigned in a meeting?)
- Communication thread (email/chat references)

### Step 5b: Knowledge Extraction

Alongside outcome tracking, scan each work item for **reusable knowledge**:

1. **Invoke the `session-knowledge` skill** with the session data already gathered
2. The skill identifies deployment procedures, environment configs, debugging patterns, architectural decisions, and new workflows
3. Knowledge is persisted as Copilot instructions, skill updates, code documentation, or store_memory facts

> **Note:** When session-outcomes is called from close-day or start-day, knowledge extraction runs automatically. For ad-hoc invocations, the user can choose to skip this step.

### Step 6: Present Summary to User

Display a consolidated outcomes report:

```markdown
## 📋 Session Outcomes Report — {Date/Range}

### Sessions Analyzed: {N} ({total_turns} turns across {total_duration})

---

### 🏢 Official Outcomes
| # | Outcome | Team | Task | Status |
|---|---------|------|------|--------|
| 1 | {title} | {team} | #{id} (existing) | Updated |
| 2 | {title} | {team} | #{id} (new) | Created |

### 🏠 Personal Outcomes
| # | Outcome | Category | Task | Status |
|---|---------|----------|------|--------|
| 1 | {title} | {category} | #{id} (new) | Created |

### ⏭️ Skipped Sessions
| Session | Reason |
|---------|--------|
| {name} | < 3 turns |
| {name} | Already processed |

---

### 📊 Summary
- **Total distinct outcomes:** {N}
- **Official:** {M} | **Personal:** {P}
- **Tasks created:** {new_count} | **Tasks updated:** {existing_count}
- **Estimated total work time:** {total_duration}
```

### Step 7: Interactive Refinement

After presenting the summary, ask:

```
ask_user:
  question: "Would you like to adjust anything?"
  choices:
    - "Looks good — no changes needed (Recommended)"
    - "Reclassify some outcomes (personal ↔ official)"
    - "Link outcomes to specific goals"
    - "Merge or split some outcomes"
    - "Remove some outcomes"
```

If the user wants changes:
- **Reclassify:** Update the task tags in Daily Planner (add/remove "official"/"personal")
- **Link to goals:** Update the Daily Planner task with a goal link
- **Merge:** Combine two work items into one task, update description
- **Split:** Create additional tasks from a single work item
- **Remove:** Delete the created task

After applying changes, show the updated summary.

---

## Idempotency & Deduplication

To prevent processing the same session twice:

1. **Before processing:** Check if the session has already been processed by searching for tasks tagged with the session ID:
   ```
   DailyPlanner-search_tasks(query: "{session_id}")
   ```

2. **After processing:** Include the session ID in task descriptions and activity logs

3. **If already processed:** Show:
   ```
   ℹ️ Session "{session_summary}" was already processed.
   Existing outcomes: {list of tasks}
   Would you like to reprocess it?
   ```

---

## Integration with Other Skills

### close-day integration
When invoked by close-day, this skill:
- Runs automatically with scope = today
- Presents outcomes as part of the day review (Step 1 of close-day)
- Keeps interaction minimal — no refinement step unless issues found
- Passes outcome data to close-day for the day summary

### start-day integration
When invoked by start-day, this skill:
- Runs automatically with scope = yesterday
- Only processes sessions not already processed
- Highlights any untracked work from yesterday
- Keeps interaction minimal — quick report only

### impact-tracker integration
When creating official outcomes:
- If a task would benefit from impact tracking, suggest:
  > "📊 This official outcome may be worth tracking for your performance review. Run the **impact-tracker** skill to document its impact."

---

## Edge Cases

| Scenario | Handling |
|----------|----------|
| Session with only 1-2 turns | Skip — too trivial to extract outcomes |
| Session with one clear task | Create single outcome, no splitting needed |
| Session already has a named task (e.g., "142-Build Email Template") | Link to existing task, don't create new |
| Work item spans multiple sessions | Note in description "Continued from session {id}" |
| No checkpoints available | Fall back to turn-by-turn analysis |
| WorkIQ/Teams unavailable | Skip enrichment, proceed with session data only |
| Daily Planner tools unavailable | **STOP — work with user to fix (Step 0.1). Do NOT fall back to Notion.** |
| User rejects all outcomes | Acknowledge and mark sessions as reviewed |
| Session is current (still active) | Skip — only process completed/idle sessions |

---

## Tools & APIs Used

### Session Store (SQLite — read-only)
- `sessions` — Session metadata, repository, branch, summary
- `turns` — Full conversation history
- `checkpoints` — Structured work summaries
- `session_files` — Files created/edited per session
- `session_refs` — Commits, PRs, issues linked to sessions
- `search_index` — Full-text search across all session data

### Daily Planner
- `DailyPlanner-search_tasks` — Find existing matching tasks
- `DailyPlanner-get_task` — Get task details
- `DailyPlanner-create_task` — Create new tasks for untracked work
- `DailyPlanner-add_activity_log` — Log outcomes to existing tasks
- `DailyPlanner-complete_task` — Mark completed work
- `DailyPlanner-update_task` — Update tags, descriptions
- `DailyPlanner-sync_copilot_sessions` — Sync sessions to backend

### External Enrichment
- `workiq-ask_work_iq` — Related emails, Teams messages, meetings
- `teams-SearchTeamMessagesQueryParameters` — Direct Teams message search
- `DailyPlanner-get_todays_meetings` — Meeting correlation
- `calendar-ListCalendarView` — Calendar event lookup

### Utility
- `ask_user` — Classification disambiguation, user preferences
- `sql` (session_store database) — Query session history

---

## Examples of User Requests That Trigger This Skill

| User Says | Action |
|-----------|--------|
| "review my sessions" | Ad-hoc: ask for date range, then process |
| "extract session outcomes" | Ad-hoc: ask for scope |
| "what did I do today" | Process today's sessions |
| "what did I do in session X" | Process specific session |
| "process sessions from this week" | Process all sessions from current week |
| "untracked work" | Find sessions with no corresponding Daily Planner tasks |
| "session work" | Ad-hoc session review |
| (from close-day) | Auto-process today's sessions |
| (from start-day) | Auto-process yesterday's sessions |

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
  description = "Surfaced by: session-outcomes · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "session-outcomes"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.