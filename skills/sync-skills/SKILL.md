---
description: "Sync Copilot CLI skills, instructions, and agents with the skills-plugin-marketplace GitHub repository. Use this skill when the user says 'sync skills', 'pull skills', 'push skills', 'download skills', 'upload skills', 'backup skills', 'restore skills', 'sync from repo', 'sync to repo', 'install skills', 'publish skills', or 'set up skills on new machine'. Handles both directions: pulls remote changes, then pushes local changes."
---

# Sync Skills

Bidirectional sync between local `~/.copilot/` and the `piusn/skills-plugin-marketplace` GitHub repository. Pulls remote updates first, then pushes any local changes — a single command to stay fully in sync.

## When to Use

- After creating or updating a skill, instruction, or agent locally
- After someone (or another machine) pushed updates to the repository
- Setting up Copilot CLI on a new machine
- Periodically syncing to stay current across machines

## ⚠️ Safety: No Deletions

This skill **never deletes files**. It only pulls additions/updates and pushes additions/modifications. To remove a file from the repo, use `git rm` manually.

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

**If not a git repo (new machine setup):**
```powershell
cd $copilotDir
git init
git remote add origin https://github.com/piusn/skills-plugin-marketplace.git
git fetch origin
git checkout -b main origin/main
```

If not a git repo and initialization succeeds, skip to Step 4 (verify) — the checkout already pulled everything.

### Step 2: Pull Remote Changes

Stash any local changes, pull, then restore:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"

# Check for local changes
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

### Step 3: Push Local Changes

Stage tracked directories, commit if there are changes, and push:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"

git -C $copilotDir add skills/ instructions/ agents/ .gitignore README.md

$staged = git -C $copilotDir diff --cached --stat
if ($staged) {
    # Summarize what changed for the commit message
    git -C $copilotDir --no-pager diff --cached --stat

    git -C $copilotDir commit -m "chore: sync skills, instructions, and agents

[Summary of changes]

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"

    git -C $copilotDir push
} else {
    Write-Host "No local changes to push"
}
```

### Step 4: Verify & Report

Count what's installed and report both directions:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
$skillCount = (Get-ChildItem "$copilotDir\skills" -Directory).Count
$instrCount = (Get-ChildItem "$copilotDir\instructions" -File -Filter "*.md").Count
$agentCount = (Get-ChildItem "$copilotDir\agents" -File -Filter "*.md" -ErrorAction SilentlyContinue).Count
```

Report results in this format:

```markdown
## ✅ Skills Synced

| Category | Count | Location |
|----------|-------|----------|
| Skills | N | ~/.copilot/skills/ |
| Instructions | N | ~/.copilot/instructions/ |
| Agents | N | ~/.copilot/agents/ |

| Direction | Result |
|-----------|--------|
| ⬇️ Pull | Already up to date / N files updated |
| ⬆️ Push | No local changes / Pushed commit abc1234 |

📂 Path: {resolved path}
🔗 Repo: https://github.com/piusn/skills-plugin-marketplace
```

## Error Handling

- **Not a git repo:** Initialize with `git init` + `git remote add` + `git fetch` + `git checkout`
- **Pull conflicts:** Stash local changes, pull --rebase, pop stash
- **Push rejected:** Pull --rebase first, then retry push
- **No remote:** Add remote: `git remote add origin https://github.com/piusn/skills-plugin-marketplace.git`
- **No changes in either direction:** Report "Already in sync" — do not create empty commits

## Notes

- `~/.copilot/` IS the git repo — no file copying needed
- Always pulls first, then pushes — this avoids push rejections
- `.gitignore` ensures only skills/, instructions/, agents/ are tracked
- Secrets (mcp-config.json), logs, packages, and session data are never committed
- Local-only files and runtime data are never affected
- Always use `$env:USERPROFILE` — never hardcode a username in paths
- On a brand new machine, the skill handles full initialization (init + clone)
