---
description: "Sync local Copilot CLI skills, instructions, and agents to the skills-plugin-marketplace GitHub repository. Use this skill when the user says 'sync skills to github', 'push skills', 'upload skills', 'backup skills', 'sync to repo', or 'publish skills'. Commits and pushes directly from ~/.copilot which is itself a git repo."
---

# Sync Skills to GitHub

## Context
The `~/.copilot/` directory is a git repository pointing to `piusn/skills-plugin-marketplace`. Skills, instructions, and agents are tracked directly — no copying needed. This skill simply stages, commits, and pushes.

## When to Use
- After creating or updating a skill, instruction, or agent locally
- When the user wants to back up their Copilot configuration
- Before switching machines, to ensure the latest versions are published

## ⚠️ Safety: No Deletions
This skill **never deletes files**. It only stages additions and modifications. To remove a file from the repo, use `git rm` manually.

## Important: Path Resolution
**NEVER hardcode a username in paths.** Always resolve dynamically:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
```

## Workflow

### Step 1: Verify Repo
Confirm `~/.copilot/` is a git repo with the correct remote:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
git -C $copilotDir remote -v
```

If not a git repo, inform the user and stop.

### Step 2: Stage Changes
Stage all tracked file changes (additions and modifications only):

```powershell
git -C $copilotDir add skills/ instructions/ agents/ .gitignore README.md
```

### Step 3: Check for Changes
```powershell
$status = git -C $copilotDir status --porcelain
```

If `$status` is empty, report "Everything is already up to date" and stop.

### Step 4: Summarize Changes
Show what changed:

```powershell
git -C $copilotDir --no-pager diff --cached --stat
```

### Step 5: Commit and Push
```powershell
git -C $copilotDir commit -m "chore: sync skills, instructions, and agents

[Summary of changes]

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"

git -C $copilotDir push
```

### Step 6: Report Results
```markdown
## ✅ Skills Synced to GitHub

| Metric | Count |
|--------|-------|
| Files changed | N |
| Commit | abc1234 |

🔗 [View on GitHub](https://github.com/piusn/skills-plugin-marketplace)
```

## Error Handling
- **Not a git repo:** Tell user to run `git init` + `git remote add origin https://github.com/piusn/skills-plugin-marketplace.git`
- **Push rejected:** Run `git -C $copilotDir pull --rebase` first, then retry push
- **No changes:** Report "Already up to date" — do not create empty commits

## Notes
- `~/.copilot/` IS the git repo — no file copying needed
- `.gitignore` ensures only skills/, instructions/, agents/ are tracked
- Secrets (mcp-config.json), logs, packages, and session data are never committed
- Always use `$env:USERPROFILE` — never hardcode a username in paths
