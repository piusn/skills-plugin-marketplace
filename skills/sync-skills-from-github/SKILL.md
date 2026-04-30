---
description: "Sync skills, instructions, and agents from the skills-plugin-marketplace GitHub repository to the local machine. Use this skill when the user says 'sync skills from github', 'pull skills', 'download skills', 'restore skills', 'sync from repo', 'install skills', or 'set up skills on new machine'. Pulls the latest from GitHub and copies into the local .copilot directory."
---

# Sync Skills from GitHub

## Context
When setting up a new machine or pulling updates made on another machine, this skill downloads the latest skills, instructions, and agents from the `skills-plugin-marketplace` GitHub repository and installs them into the local `~/.copilot/` directory.

## When to Use
- Setting up Copilot CLI on a new machine
- After someone (or another machine) pushed updates to the repository
- Restoring skills after a reinstall or profile reset
- Periodically pulling to stay in sync

## ⚠️ Safety: No Deletions
**This skill NEVER deletes files from the local disk.** It only adds new files and updates existing ones.

- If a skill exists locally but not in the repo, it is **left on disk** (it may be a work-in-progress or machine-specific)
- If a skill exists in the repo but not locally, it is **added to disk**
- If a skill exists in both places, the **repo version overwrites the local version** (repo is the source of truth for this direction)

To explicitly remove a local skill, delete it manually with `Remove-Item`.

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
Determine the repository and local target paths:

```powershell
$copilotDir = Join-Path $env:USERPROFILE ".copilot"
$repoDir = "<repository-working-directory>"  # The current working directory of the repo
```

**Standard clone location:** `C:\repositories\skills-plugin-marketplace`
**Clone URL:** `https://github.com/piusn/skills-plugin-marketplace.git`

If the repo is not found at the current working directory, check the standard location above. If it doesn't exist anywhere, clone it first:
```powershell
git clone https://github.com/piusn/skills-plugin-marketplace.git "C:\repositories\skills-plugin-marketplace"
```

Verify the repo exists and has a remote:
```powershell
git -C $repoDir remote -v
```

### Step 2: Pull Latest from GitHub
Ensure the local repository is up to date:

```powershell
git -C $repoDir pull --rebase
```

If the pull fails due to local changes, warn the user and suggest resolving conflicts first.

### Step 3: Ensure Target Directories Exist
Create the target directories if they don't exist (new machine scenario):

```powershell
$skillsTarget = Join-Path $copilotDir "skills"
$instrTarget = Join-Path $copilotDir "instructions"
$agentsTarget = Join-Path $copilotDir "agents"

New-Item -ItemType Directory -Path $skillsTarget -Force | Out-Null
New-Item -ItemType Directory -Path $instrTarget -Force | Out-Null
New-Item -ItemType Directory -Path $agentsTarget -Force | Out-Null
```

### Step 4: Sync Skills (additive — no deletions)
Copy skill directories from repo to local. Only adds new and updates existing — never deletes local-only skills:

```powershell
$skillsSource = Join-Path $repoDir "skills"

# DO NOT delete local skills that are missing from repo — they may be WIP or machine-specific
# Only add/update skills from repo
Get-ChildItem $skillsSource -Directory | ForEach-Object {
    Copy-Item $_.FullName "$skillsTarget\$($_.Name)" -Recurse -Force
}
```

### Step 5: Sync Instructions (additive — no deletions)
Copy instruction files from repo to local. Only adds new and updates existing — never deletes local-only instructions:

```powershell
$instrSource = Join-Path $repoDir "instructions"

# DO NOT delete local instructions not in repo
# Only add/update instructions from repo
Copy-Item "$instrSource\*.md" $instrTarget -Force
```

### Step 6: Sync Agents (additive — no deletions)
Copy agent files from repo to local. Only adds new and updates existing — never deletes local-only agents:

```powershell
$agentsSource = Join-Path $repoDir "agents"

# DO NOT delete local agents not in repo
# Only add/update agents from repo
Copy-Item "$agentsSource\*.md" $agentsTarget -Force
```

### Step 7: Verify Installation
Count what was installed and verify key files exist:

```powershell
$skillCount = (Get-ChildItem $skillsTarget -Directory).Count
$instrCount = (Get-ChildItem $instrTarget -File -Filter "*.md").Count
$agentCount = (Get-ChildItem $agentsTarget -File -Filter "*.md").Count
```

### Step 8: Report Results
Present an installation summary:

```markdown
## ✅ Skills Synced from GitHub

| Category | Count | Location |
|----------|-------|----------|
| Skills | N | ~/.copilot/skills/ |
| Instructions | N | ~/.copilot/instructions/ |
| Agents | N | ~/.copilot/agents/ |

📂 Installed to: [resolved copilot directory path]
🔗 Source: [repository remote URL]
```

## Error Handling
- **Repository not cloned:** Suggest cloning first: `git clone https://github.com/piusn/skills-plugin-marketplace.git`
- **Pull conflicts:** Warn the user and suggest resolving manually or force-pulling with `git reset --hard origin/main`
- **Source directory missing in repo:** Skip that category with a warning (e.g., "No agents/ directory in repo — skipping")
- **Permission denied:** Suggest running with elevated permissions if target directory is restricted
- **No .copilot directory:** Create it — this is the new machine scenario

## Tools & APIs Used
- `powershell` — File operations and git commands
- `git` — Version control operations

## Output Format
Summary table showing items installed and the resolved local path.

## Notes
- **No deletions** — this skill only adds and updates, never removes files from the local disk
- To remove a local skill, delete it manually with `Remove-Item`
- Items that exist locally but not in the repo are left untouched (they may be WIP or machine-specific)
- Always use `$env:USERPROFILE` or `$HOME` — never hardcode a username in paths
- On a brand new machine, the `.copilot` directory and subdirectories are created automatically
- The skill pulls from git first to ensure it's working with the latest version
