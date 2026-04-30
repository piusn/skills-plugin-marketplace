---
description: "Sync local Copilot CLI skills, instructions, and agents to the skills-plugin-marketplace GitHub repository. Use this skill when the user says 'sync skills to github', 'push skills', 'upload skills', 'backup skills', 'sync to repo', or 'publish skills'. Copies from the local .copilot directory, commits changes, and pushes to the remote repository."
---

# Sync Skills to GitHub

## Context
Skills, instructions, and agents are authored locally in the user's `~/.copilot/` directory. This skill copies them into the `skills-plugin-marketplace` repository, commits the changes, and pushes to GitHub — keeping the remote repository up to date as a portable backup and sharing mechanism.

## When to Use
- After creating or updating a skill, instruction, or agent locally
- When the user wants to back up their Copilot configuration
- Before switching machines, to ensure the latest versions are published

## ⚠️ Safety: No Deletions
**This skill NEVER deletes files from disk or from the repository.** It only adds new files and updates existing ones.

- If a skill exists in the repo but not locally, it is **left in the repo** (it may have been created on another machine)
- If a skill exists locally but not in the repo, it is **added to the repo**
- If a skill exists in both places, the **local version overwrites the repo version** (local is the source of truth for this direction)

To explicitly remove a skill from the repo, do it manually with `git rm`.

## Important: Path Resolution
**NEVER hardcode a username in paths.** Always resolve the user's home directory dynamically:

- **PowerShell:** Use `$env:USERPROFILE` (Windows) or `$HOME` (cross-platform)
- The `.copilot` directory is always at `<home>/.copilot/`

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
# Results in e.g. C:\Users\pingugi\.copilot on Windows
```

## Workflow

### Step 1: Resolve Paths
Determine the local source and repository target paths:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
$repoDir = "<repository-working-directory>"  # The current working directory of the repo
```

Verify the repository exists and has a git remote:
```powershell
git -C $repoDir remote -v
```

### Step 2: Copy Skills (additive — no deletions)
Copy all local skill directories into the repo, overwriting existing ones but leaving repo-only skills untouched:

```powershell
$skillsSource = Join-Path $copilotDir "skills"
$skillsTarget = Join-Path $repoDir "skills"

# DO NOT delete existing repo skills — only add/update from local
Get-ChildItem $skillsSource -Directory | ForEach-Object {
    Copy-Item $_.FullName "$skillsTarget\$($_.Name)" -Recurse -Force
}
```

### Step 3: Copy Instructions (additive — no deletions)
Copy all local instruction files, leaving repo-only instructions untouched:

```powershell
$instrSource = Join-Path $copilotDir "instructions"
$instrTarget = Join-Path $repoDir "instructions"

# DO NOT delete existing repo instructions — only add/update from local
Copy-Item "$instrSource\*.md" $instrTarget -Force
```

### Step 4: Copy Agents (additive — no deletions)
Copy all local agent files, leaving repo-only agents untouched:

```powershell
$agentsSource = Join-Path $copilotDir "agents"
$agentsTarget = Join-Path $repoDir "agents"

# DO NOT delete existing repo agents — only add/update from local
Copy-Item "$agentsSource\*.md" $agentsTarget -Force
```

### Step 5: Check for Changes
Before committing, check if anything actually changed:

```powershell
git -C $repoDir add -A
$status = git -C $repoDir status --porcelain
```

If `$status` is empty, report "Everything is already up to date" and stop.

### Step 6: Summarize Changes
Show the user what changed before committing:

```powershell
git -C $repoDir --no-pager diff --cached --stat
```

Present a summary table:

| Type | Added | Modified | Deleted |
|------|-------|----------|---------|
| Skills | N | N | 0 |
| Instructions | N | N | 0 |
| Agents | N | N | 0 |

### Step 7: Commit and Push
Commit with a descriptive message summarizing what changed:

```powershell
git -C $repoDir commit -m "chore: sync skills, instructions, and agents

[Summary of changes — e.g., added 2 skills, updated 3 instructions]

Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"

git -C $repoDir push
```

### Step 8: Report Results
Present a completion summary:

```markdown
## ✅ Skills Synced to GitHub

| Metric | Count |
|--------|-------|
| Skills | 47 |
| Instructions | 17 |
| Agents | 1 |
| Files changed | N |
| Commit | abc1234 |

🔗 [View on GitHub](https://github.com/piusn/skills-plugin-marketplace)
```

## Error Handling
- **No git remote:** Warn the user and suggest running `git remote add origin <url>`
- **Push rejected:** Suggest `git pull --rebase` first, then retry push
- **Source directory missing:** Skip that category with a warning (e.g., "No agents directory found — skipping")
- **No changes detected:** Report "Already up to date" — do not create empty commits

## Tools & APIs Used
- `powershell` — File operations and git commands
- `git` — Version control operations

## Output Format
Summary table showing files synced, changes committed, and a link to the GitHub repository.

## Notes
- **No deletions** — this skill only adds and updates, never removes files from the repo
- To remove a skill from the repo, use `git rm` manually
- Always use `$env:USERPROFILE` or `$HOME` — never hardcode a username in paths
- The commit message should summarize what actually changed, not be generic
