---
name: sync-copilot-data
description: >
  Sync Copilot sessions, skills, and instructions to the Daily Planner backend
  for centralized viewing at kamin.day. Use this skill when the user says
  'sync copilot data', 'sync copilot sessions', 'sync copilot skills',
  'sync copilot instructions', 'push copilot data', 'sync copilot to daily planner',
  or 'sync copilot everything'. Runs all three sync tools and reports results.
---

# Sync Copilot Data — Push Sessions, Skills & Instructions to Daily Planner

Syncs local Copilot CLI data (sessions, skills, instructions) to the Daily Planner backend so it's viewable from any machine at kamin.day.

## Why This Skill Exists

Copilot sessions, skills, and instructions live locally on each machine. This skill centralizes them in the Daily Planner backend, making them accessible from any browser. It's the on-demand version of the sync that runs automatically during `close-day` and `start-day`.

## When to Use

- After a productive session you want to capture immediately
- When switching machines and want the latest data available
- After creating or updating skills/instructions
- Periodically throughout the day to keep the backend current
- When you just want to make sure everything is synced

## Workflow

### Step 1: Ask What to Sync

```
ask_user:
  question: "What should I sync to the Daily Planner backend?"
  choices:
    - "Everything — sessions, skills, and instructions (Recommended)"
    - "Sessions only"
    - "Skills only"
    - "Instructions only"
    - "Skills + Instructions (no sessions)"
```

### Step 2: Sync Sessions (if selected)

```
DailyPlanner-sync_copilot_sessions(since: "{appropriate_date}")
```

Default `since` behavior:
- If time is morning (before noon): use yesterday's date
- If time is afternoon/evening: use today's date
- If user specifies a date range, use that instead

Report:
```
✅ Sessions: {created} new, {updated} updated ({total} total synced)
```

If errors occurred, report them:
```
⚠️ Sessions: {created} new, {updated} updated, {error_count} errors
   Errors: {error_details}
```

### Step 3: Sync Skills (if selected)

```
DailyPlanner-sync_copilot_skills()
```

Report:
```
✅ Skills: {created} new, {updated} updated
```

### Step 4: Sync Instructions (if selected)

Sync user-level instructions from `~/.copilot/instructions/`:

```
DailyPlanner-sync_copilot_instructions()
```

Optionally, if the current working directory is a git repository with `.github/instructions/`, also sync repo-level instructions:

```
DailyPlanner-sync_copilot_instructions(repositoryPaths: "{cwd}")
```

Report:
```
✅ Instructions: {created} new, {updated} updated
```

### Step 5: Summary Report

Present a clean summary:

```markdown
## ✅ Sync Complete

| Data | New | Updated | Errors |
|------|-----|---------|--------|
| Sessions | {N} | {M} | {E} |
| Skills | {N} | {M} | {E} |
| Instructions | {N} | {M} | {E} |

🌐 View at: https://kamin.day
- Sessions: https://kamin.day/copilot-sessions
- Skills: https://kamin.day/copilot-skills
- Instructions: https://kamin.day/copilot-instructions
```

## Tools Used

- `DailyPlanner-sync_copilot_sessions` — Reads local SQLite, pushes sessions to backend
- `DailyPlanner-sync_copilot_skills` — Reads `~/.copilot/skills/`, pushes to backend
- `DailyPlanner-sync_copilot_instructions` — Reads `~/.copilot/instructions/` + repo instructions, pushes to backend
- `ask_user` — Scope selection

## Notes

- All syncs are idempotent — running multiple times is safe (upserts by unique key)
- Sessions are deduped by their original session UUID
- Skills are deduped by name
- Instructions are deduped by name + source + repository (compound key)
- This skill does NOT sync to GitHub — use `sync-skills-to-github` for that
- For a full backup, run both this skill and `sync-skills-to-github`
