---
description: "Sync skills, instructions, and agents from the skills-plugin-marketplace GitHub repository to the local machine. Use this skill when the user says 'sync skills from github', 'pull skills', 'download skills', 'restore skills', 'sync from repo', 'install skills', or 'set up skills on new machine'. Pulls the latest from GitHub directly into ~/.copilot."
---

# Sync Skills from GitHub

## Context
The `~/.copilot/` directory is a git repository pointing to `piusn/skills-plugin-marketplace`. Pulling from GitHub updates skills, instructions, and agents in place — no copying needed.

## When to Use
- Setting up Copilot CLI on a new machine
- After someone (or another machine) pushed updates to the repository
- Restoring skills after a reinstall or profile reset
- Periodically pulling to stay in sync

## ⚠️ Safety: No Deletions
This skill **never deletes local files**. It only pulls additions and updates from the remote. Local-only files are left untouched.

To remove a local skill, delete it manually with `Remove-Item`.

## Important: Path Resolution
**NEVER hardcode a username in paths.** Always resolve dynamically:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
```

## Workflow

### Step 1: Verify Repo
Confirm `~/.copilot/` is a git repo:

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

### Step 2: Pull Latest
```powershell
git -C $copilotDir pull --rebase
```

If the pull fails due to local changes, warn the user and suggest:
```powershell
git -C $copilotDir stash
git -C $copilotDir pull --rebase
git -C $copilotDir stash pop
```

### Step 3: Verify Installation
Count what's installed:

```powershell
$skillCount = (Get-ChildItem "$copilotDir\skills" -Directory).Count
$instrCount = (Get-ChildItem "$copilotDir\instructions" -File -Filter "*.md").Count
$agentCount = (Get-ChildItem "$copilotDir\agents" -File -Filter "*.md" -ErrorAction SilentlyContinue).Count
```

### Step 4: Report Results
```markdown
## ✅ Skills Synced from GitHub

| Category | Count | Location |
|----------|-------|----------|
| Skills | N | ~/.copilot/skills/ |
| Instructions | N | ~/.copilot/instructions/ |
| Agents | N | ~/.copilot/agents/ |

📂 Installed to: {resolved path}
🔗 Source: https://github.com/piusn/skills-plugin-marketplace
```

## Error Handling
- **Not a git repo:** Initialize with `git init` + `git remote add` + `git fetch` + `git checkout`
- **Pull conflicts:** Stash local changes, pull, then pop stash
- **No remote:** Add remote: `git remote add origin https://github.com/piusn/skills-plugin-marketplace.git`

## Notes
- `~/.copilot/` IS the git repo — no file copying needed
- `git pull` only updates tracked files (skills/, instructions/, agents/)
- Local-only files and runtime data are never affected (protected by .gitignore)
- Always use `$env:USERPROFILE` — never hardcode a username in paths
- On a brand new machine, the skill handles full initialization (init + clone)
