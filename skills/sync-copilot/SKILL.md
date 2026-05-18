---
name: sync-copilot
description: >
  One-command sync that keeps the Copilot CLI fully up to date in both
  directions: (1) bidirectional git sync between `~/.copilot/` and the
  `piusn/skills-plugin-marketplace` GitHub repo (skills, instructions,
  agents), and (2) push local Copilot sessions, skills, and instructions
  to the Daily Planner backend so they're viewable at kamin.day. Use this
  skill when the user says 'sync', 'sync copilot', 'sync everything',
  'sync skills', 'sync copilot data', 'sync copilot sessions', 'pull skills',
  'push skills', 'backup copilot', 'restore copilot', or 'set up copilot on
  new machine'. Replaces the legacy `sync-skills` and `sync-copilot-data`
  skills.
---

# Sync Copilot — GitHub Marketplace + Daily Planner Backend

Single command that:

1. **GitHub side:** pulls remote updates from `piusn/skills-plugin-marketplace`, then pushes any local changes (`~/.copilot/skills/`, `instructions/`, `agents/`).
2. **Backend side:** pushes local Copilot sessions, skills, and instructions to the Daily Planner backend so they're viewable at https://kamin.day from any machine.

Replaces `sync-skills` (GitHub-only) and `sync-copilot-data` (backend-only). Run this skill any time you want everything in sync.

## When to Use

- After creating or updating a skill, instruction, or agent
- After a productive session you want to capture immediately
- When switching machines and want the latest data available
- Setting up Copilot CLI on a new machine (full bootstrap)
- Periodically throughout the day to keep both stores current

## ⚠️ Safety: No Deletions

This skill **never deletes files**. It only pulls additions/updates and pushes additions/modifications, both on GitHub and the Daily Planner backend. To remove a file from the repo, use `git rm` manually. To remove a session/skill/instruction from the backend, use its delete endpoint directly.

## Important: Path Resolution

**NEVER hardcode a username in paths.** Always resolve dynamically:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
```

## Workflow

### Step 0: Decide Scope

By default, sync **everything** — both GitHub and the Daily Planner backend.

If the user's phrasing is narrower (e.g., "sync skills to github", "sync sessions only"), respect that. Otherwise, do not prompt — just run the full sync.

Optional scope choices (only ask if the user signals ambiguity):

```
ask_user:
  question: "What should I sync?"
  choices:
    - "Everything — GitHub + Daily Planner backend (Recommended)"
    - "GitHub only (skills + instructions + agents)"
    - "Daily Planner backend only (sessions + skills + instructions)"
    - "Sessions only (to backend)"
```

---

### Phase A — GitHub Sync (`~/.copilot/` ↔ skills-plugin-marketplace)

#### A.1: Verify Repo

Confirm `~/.copilot/` is a git repo with the correct remote:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
git -C $copilotDir remote -v
```

**If not a git repo (new machine setup):**

```powershell
cd $copilotDir
git init
git remote add origin https://github.com/piusn/skills-plugin-marketplace.git
git fetch origin
git checkout -b main origin/main
```

If initialization succeeds on a new machine, skip A.2/A.3 — the `checkout` already pulled everything. Continue to Phase B.

#### A.2: Pull Remote Changes

Stash any local changes, pull, then restore:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
$dirty = git -C $copilotDir status --porcelain

if ($dirty) {
    git -C $copilotDir stash
    git -C $copilotDir pull --rebase
    git -C $copilotDir stash pop
} else {
    git -C $copilotDir pull --rebase
}
```

Record pull result (e.g., "Already up to date" or files changed).

#### A.3: Push Local Changes

Stage tracked directories, commit if there are changes, and push:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"

git -C $copilotDir add skills/ instructions/ agents/ .gitignore README.md

$staged = git -C $copilotDir diff --cached --stat
if ($staged) {
    git -C $copilotDir --no-pager diff --cached --stat
    git -C $copilotDir commit -m "chore: sync skills, instructions, and agents

[Summary of changes]

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
    git -C $copilotDir push
} else {
    Write-Host "No local changes to push"
}
```

---

### Phase B — Daily Planner Backend Sync

#### B.1: Sessions

First, determine the last sync date by querying the backend for the most recently synced session:

```
DailyPlanner-get_copilot_sessions()
```

Look at the most recent session's `Updated` or `Created` date. Use that as the `since` parameter. If no sessions exist in the backend, omit `since` to do a full sync.

```
DailyPlanner-sync_copilot_sessions(since: "{last_sync_date}")
```

**Important:** Do NOT use morning/afternoon heuristics or hardcoded dates. Always derive the `since` date from the backend's most recent session. This ensures no sessions are missed and no unnecessary re-syncing occurs.

#### B.2: Skills

```
DailyPlanner-sync_copilot_skills()
```

#### B.3: Instructions

Sync user-level instructions from `~/.copilot/instructions/`:

```
DailyPlanner-sync_copilot_instructions()
```

Optionally, if the current working directory is a git repository with `.github/instructions/`, also sync repo-level instructions:

```
DailyPlanner-sync_copilot_instructions(repositoryPaths: "{cwd}")
```

---

### Step C — Verify & Report

Count what's installed locally and report both directions across both phases:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
$skillCount = (Get-ChildItem "$copilotDir\skills" -Directory).Count
$instrCount = (Get-ChildItem "$copilotDir\instructions" -File -Filter "*.md").Count
$agentCount = (Get-ChildItem "$copilotDir\agents" -File -Filter "*.md" -ErrorAction SilentlyContinue).Count
```

Report results in this unified format:

```markdown
## ✅ Sync Complete

### Local Inventory

| Category | Count | Location |
|----------|-------|----------|
| Skills | N | ~/.copilot/skills/ |
| Instructions | N | ~/.copilot/instructions/ |
| Agents | N | ~/.copilot/agents/ |

### GitHub (skills-plugin-marketplace)

| Direction | Result |
|-----------|--------|
| ⬇️ Pull | Already up to date / N files updated |
| ⬆️ Push | No local changes / Pushed commit abc1234 |

### Daily Planner Backend (kamin.day)

| Data | New | Updated | Errors |
|------|-----|---------|--------|
| Sessions | N | M | E |
| Skills | N | M | E |
| Instructions | N | M | E |

📂 Local path: {resolved path}
🔗 GitHub: https://github.com/piusn/skills-plugin-marketplace
🌐 Backend:
  - Sessions: https://kamin.day/copilot-sessions
  - Skills: https://kamin.day/copilot-skills
  - Instructions: https://kamin.day/copilot-instructions
```

Omit any section the user explicitly opted out of in Step 0.

## Tools Used

- `git` (via PowerShell) — for the GitHub-side bidirectional sync
- `DailyPlanner-get_copilot_sessions` — read backend's latest session to compute `since`
- `DailyPlanner-sync_copilot_sessions` — push local SQLite sessions to backend
- `DailyPlanner-sync_copilot_skills` — push `~/.copilot/skills/` to backend
- `DailyPlanner-sync_copilot_instructions` — push `~/.copilot/instructions/` + repo instructions to backend
- `ask_user` — scope selection (only when ambiguous)

## Error Handling

- **GitHub: Not a git repo** — Initialize with `git init` + `git remote add` + `git fetch` + `git checkout` (see A.1).
- **GitHub: Pull conflicts** — Stash local changes, pull --rebase, pop stash.
- **GitHub: Push rejected** — Pull --rebase first, then retry push.
- **GitHub: No remote** — Add remote: `git remote add origin https://github.com/piusn/skills-plugin-marketplace.git`.
- **GitHub: No changes in either direction** — Report "Already in sync" — do not create empty commits.
- **Backend: tool errors** — Surface errors in the summary table's `Errors` column; do not abort the remaining steps. Phase A and Phase B are independent — failure in one should not block the other.

## Notes

- `~/.copilot/` IS the git repo — no file copying needed.
- Always pulls from GitHub before pushing — avoids push rejections.
- `.gitignore` ensures only `skills/`, `instructions/`, `agents/` are tracked. Secrets (`mcp-config.json`), logs, packages, and session data are never committed to GitHub.
- All backend syncs are idempotent (upserts by unique key): sessions by UUID, skills by name, instructions by name+source+repository.
- Phase A and Phase B are independent — one can succeed while the other fails. The skill always reports both outcomes.
- On a brand new machine, the skill handles full initialization (git init + clone). After that, run again to also populate the backend.
- This is the consolidated successor to the now-removed `sync-skills` and `sync-copilot-data` skills.
