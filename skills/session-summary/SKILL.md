---
name: session-summary
description: >
  Run all session wrap-up skills in one go: extract outcomes, capture knowledge,
  sync sessions, and log activity. Use this skill when the user says "session summary",
  "wrap up session", "summarize session", "summarize the session", "summarise sessions",
  "process this session", "session recap", "what did we do", "what did we accomplish",
  "end session", "close session", or "session wrapup". Orchestrates session-outcomes
  and session-knowledge so you don't have to call them individually.
---

# Session Summary — One-Shot Session Wrap-Up

Orchestrates all session-based skills in the correct order to fully process a Copilot session: verify tools, sync sessions, extract outcomes, capture knowledge, and present a unified summary.

## Why This Skill Exists

Wrapping up a session previously required invoking `session-outcomes` and `session-knowledge` separately. This skill runs them in sequence with proper dependency ordering and a single summary at the end.

## When to Use

- End of a work session ("wrap up", "session summary")
- Before switching to a different task ("summarize what we did")
- Ad-hoc review ("what did we do today")
- Replaces calling `session-outcomes` and `session-knowledge` individually

## Workflow

### Step 1: Verify Daily Planner Tools

**CRITICAL: Stop if Daily Planner tools are unavailable.**

Test with:
```
DailyPlanner-search_tasks(query: "test")
```

If the call fails:
```
ask_user:
  question: "Daily Planner MCP tools are not available. We need them to track outcomes. How should we proceed?"
  choices:
    - "I'll restart the session and re-invoke"
    - "Kill and restart the Daily Planner MCP server"
    - "Skip — just show the summary without tracking"
```

If "Kill and restart":
1. Check if the SSE server is running: `Invoke-RestMethod -Uri "http://localhost:5101/health"`
2. If not running, start it: `Start-Process -FilePath "dotnet" -ArgumentList "run","--","--http" -WorkingDirectory "C:\repositories\Sokokapu-Limited\daily-planner-mcp\src\DailyPlannerMcp" -WindowStyle Hidden`
3. Wait 10 seconds and retry the health check
4. If still failing, ask user to restart the Copilot session

### Step 2: Sync Sessions to Backend

```
DailyPlanner-sync_copilot_sessions(since: "{today_or_scope_date}")
```

Report sync count briefly and continue.

### Step 3: Extract Outcomes (session-outcomes)

Follow the `session-outcomes` skill workflow:

1. **Gather session data** from the current conversation context:
   - Commits made (from git log)
   - Files created/modified
   - Key decisions and work items

2. **Extract distinct work items** — each logically separate effort

3. **Classify** as personal or official

4. **Match or create Daily Planner tasks** for each outcome

5. **Mark completed tasks** where the work is done

Present the outcomes table to the user.

### Step 4: Capture Knowledge (session-knowledge)

Follow the `session-knowledge` skill workflow:

1. **Scan for knowledge signals** — mistakes, gotchas, patterns, architecture decisions, debugging steps

2. **Route each nugget** using the destination hierarchy:
   - Code README.md (closest to the code)
   - Repo instructions (.github/instructions/)
   - Global instructions (~/.copilot/instructions/)
   - Skill updates
   - store_memory (short gotcha facts)

3. **Present findings** and persist after user confirmation

### Step 5: Unified Summary

Present a single consolidated report:

```markdown
## 📋 Session Summary

### Outcomes Tracked
| # | Outcome | Task # | Status |
|---|---------|--------|--------|
| 1 | {title} | #{id} | ✅ Completed |
| 2 | {title} | #{id} | ✅ Completed |

### Knowledge Captured
| # | Knowledge | Destination | Status |
|---|-----------|-------------|--------|
| 1 | {title} | {file/memory} | ✅ Persisted |
| 2 | {title} | {file/memory} | ✅ Persisted |

### Session Stats
- **Duration:** ~{hours} hours
- **Commits:** {count}
- **Files changed:** {count}
- **Tasks tracked:** {count}
- **Knowledge items:** {count}
```

## Shortcuts

If the user says any of these, run the full workflow:

| User Says | Action |
|-----------|--------|
| "session summary" | Full workflow (Steps 1-5) |
| "wrap up session" | Full workflow |
| "summarize session" | Full workflow |
| "summarize the session" | Full workflow |
| "summarise sessions" | Full workflow |
| "process this session" | Full workflow |
| "session recap" | Full workflow |
| "what did we do" | Full workflow |
| "what did we accomplish" | Full workflow |
| "end session" | Full workflow |
| "close session" | Full workflow |
| "session wrapup" | Full workflow |

## Notes

- This skill is the **preferred** way to end a session. Use it instead of calling session-outcomes and session-knowledge separately.
- If the session is very short (< 5 turns, no commits), skip outcomes and just run knowledge capture.
- If Daily Planner tools fail mid-workflow, save progress and report what was completed vs what failed.
- The skill uses the **current conversation context** — no need to query the session store for the active session.
