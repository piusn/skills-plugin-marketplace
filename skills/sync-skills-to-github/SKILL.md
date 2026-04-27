---
description: "Sync local Copilot CLI skills, instructions, and agents to the skills-plugin-marketplace GitHub repository. Use this skill when the user says 'sync skills to github', 'push skills', 'upload skills', 'backup skills', 'sync to repo', or 'publish skills'. Makes GitHub an exact mirror of ~/.copilot/. Local .copilot/ is the source of truth."
---

# Sync Skills to GitHub

## Purpose
**Direction: Local `.copilot/` → GitHub**
**Source of truth: `.copilot/`**

Make the `skills-plugin-marketplace` GitHub repository an exact mirror of the local `~/.copilot/` directory. After this skill runs, GitHub matches what's in `.copilot/` — items are added, updated, AND removed.

> **Counterpart skill:** `sync-skills-from-github` does the reverse — GitHub is the source of truth, and `.copilot/` is made to match it.

## When to Use
- After creating or updating a skill, instruction, or agent locally
- When the user wants to back up their Copilot configuration
- Before switching machines, to ensure the latest versions are published

## Important: Path Resolution
**NEVER hardcode a username in paths.** Always resolve dynamically:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
```

## Repository Location
**Known location:** `C:\personal\skills-plugin-marketplace`
**Clone URL:** `https://github.com/piusn/skills-plugin-marketplace.git`

If not found, search common locations then ask the user:
```powershell
$candidates = @(
    "C:\personal\skills-plugin-marketplace",
    (Join-Path $env:USERPROFILE "skills-plugin-marketplace"),
    (Join-Path $env:USERPROFILE "repos\skills-plugin-marketplace"),
    (Join-Path $env:USERPROFILE "source\repos\skills-plugin-marketplace")
)
$repoDir = $candidates | Where-Object { Test-Path (Join-Path $_ ".git") } | Select-Object -First 1
```

## Workflow

### Step 1: Pull Latest First
Always pull before pushing to avoid conflicts:

```powershell
git -C $repoDir pull
```

If pull fails, warn the user and suggest `git stash` or `git reset --hard origin/main`.

### Step 2: Mirror Skills (.copilot/ is Source of Truth)
Clean the repo's skills directory and copy everything from local. This handles additions, updates, renames, and deletions in one step.

⚠️ **CRITICAL: Use content-level copy to avoid directory nesting.**

```powershell
$skillsSource = Join-Path $copilotDir "skills"
$skillsTarget = Join-Path $repoDir "skills"

# Remove all skill directories in repo (clean slate)
Get-ChildItem $skillsTarget -Directory | Remove-Item -Recurse -Force -ErrorAction SilentlyContinue

# Copy each skill's CONTENTS into a fresh directory in the repo
Get-ChildItem $skillsSource -Directory | ForEach-Object {
    $targetDir = Join-Path $skillsTarget $_.Name
    New-Item -ItemType Directory -Path $targetDir -Force | Out-Null
    Copy-Item "$($_.FullName)\*" $targetDir -Recurse -Force
}

# Remove zip files and temporary artifacts
Get-ChildItem $skillsTarget -Filter "*.zip" -Recurse | Remove-Item -Force -ErrorAction SilentlyContinue
```

> **Why `Copy-Item "$($_.FullName)\*"` instead of `Copy-Item $_.FullName`?**
> `Copy-Item <folder> <existing-folder> -Recurse` creates a nested subfolder: `skills/close-day/close-day/SKILL.md`.
> `Copy-Item <folder>\* <existing-folder> -Recurse` copies the CONTENTS into the target — correct behavior.

### Step 3: Mirror Instructions
```powershell
$instrSource = Join-Path $copilotDir "instructions"
$instrTarget = Join-Path $repoDir "instructions"

Get-ChildItem $instrTarget -Filter "*.md" | Remove-Item -Force -ErrorAction SilentlyContinue
Copy-Item "$instrSource\*.md" $instrTarget -Force
```

### Step 4: Mirror Agents
```powershell
$agentsSource = Join-Path $copilotDir "agents"
$agentsTarget = Join-Path $repoDir "agents"

if (Test-Path $agentsSource) {
    Get-ChildItem $agentsTarget -Filter "*.md" -ErrorAction SilentlyContinue | Remove-Item -Force -ErrorAction SilentlyContinue
    Copy-Item "$agentsSource\*.md" $agentsTarget -Force
}
```

### Step 5: Check for Changes
```powershell
git -C $repoDir add -A
$status = git -C $repoDir status --porcelain
```

If `$status` is empty, report "Everything is already up to date" and stop. Do not create empty commits.

### Step 6: Summarize Changes
```powershell
git -C $repoDir --no-pager diff --cached --stat
```

Present a summary table:

| Type | Added | Modified | Deleted |
|------|-------|----------|---------|
| Skills | N | N | N |
| Instructions | N | N | N |
| Agents | N | N | N |

### Step 7: Commit and Push
```powershell
git -C $repoDir commit -m "chore: sync skills, instructions, and agents

[Summary of changes — e.g., added 2 skills, updated 3 instructions, removed 1 agent]

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"

git -C $repoDir push
```

### Step 8: Report Results
```markdown
## ✅ Skills Synced to GitHub

| Metric | Count |
|--------|-------|
| Skills | N |
| Instructions | N |
| Agents | N |
| Files changed | N |
| Commit | abc1234 |

🔗 [View on GitHub](https://github.com/piusn/skills-plugin-marketplace)
```

## Error Handling
- **Repository not found:** Search known locations, then ask the user for the path.
- **No git remote:** Warn and suggest `git remote add origin <url>`.
- **Push rejected:** Suggest `git pull --rebase` first, then retry push.
- **Source directory missing:** Skip that category with a warning.
- **No changes detected:** Report "Already up to date" — do not create empty commits.

## Key Design Decisions

### Full mirror (.copilot/ is source of truth)
GitHub is made to exactly match `.copilot/`. Items deleted locally are deleted from the repo. The repo directories are cleaned before copy to handle renames and deletions cleanly.

### Pull before push
Always `git pull` before copying files. This prevents force-push situations and ensures you're building on the latest remote state. If there are conflicts, the user resolves them before the sync proceeds.

### Content-level copy (not folder copy)
Always use `Copy-Item "$source\*" $target` instead of `Copy-Item $source $target` when the target folder already exists. This prevents the PowerShell nesting bug where `Copy-Item <folder> <existing-folder> -Recurse` creates `target/foldername/foldername/`.

## Tools & APIs Used
- `powershell` — File operations and git commands
- `git` — Version control operations

## Notes
- Zip files and temporary artifacts in the skills directory are NOT copied
- Always use `$env:USERPROFILE` or `$HOME` — never hardcode a username
- The commit message should summarize what actually changed, not be generic
- This skill modifies the repo AND pushes — it is a write operation
