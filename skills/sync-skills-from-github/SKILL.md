---
description: "Sync skills, instructions, and agents from the skills-plugin-marketplace GitHub repository to the local .copilot directory. Use this skill when the user says 'sync skills from github', 'pull skills', 'download skills', 'restore skills', 'sync from repo', 'install skills', or 'set up skills on new machine'. Pulls the latest from GitHub and makes ~/.copilot/ an exact mirror. GitHub is the source of truth."
---

# Sync Skills from GitHub

## Purpose
**Direction: GitHub → Local `.copilot/`**
**Source of truth: GitHub**

Make the local `~/.copilot/` directory an exact mirror of the `skills-plugin-marketplace` GitHub repository. After this skill runs, local skills, instructions, and agents match what's in GitHub — items are added, updated, AND removed to stay in sync.

> **Counterpart skill:** `sync-skills-to-github` does the reverse — `.copilot/` is the source of truth, and GitHub is made to match it.

## When to Use
- Setting up Copilot CLI on a new machine
- After someone (or another machine) pushed updates to the repository
- Restoring skills after a reinstall or profile reset
- Periodically pulling to stay in sync

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

If not found anywhere, clone it:
```powershell
git clone https://github.com/piusn/skills-plugin-marketplace.git "C:\personal\skills-plugin-marketplace"
```

## Workflow

### Step 1: Pull Latest from GitHub
```powershell
git -C $repoDir pull
```
If pull fails due to local changes, warn the user and suggest `git stash` or `git reset --hard origin/main`.

### Step 2: Ensure Local Target Directories Exist
```powershell
@("skills", "instructions", "agents") | ForEach-Object {
    New-Item -ItemType Directory -Path (Join-Path $copilotDir $_) -Force | Out-Null
}
```

### Step 3: Sync Skills (Full Mirror — GitHub is Source of Truth)
GitHub wins. Add new skills, update changed skills, remove skills that no longer exist in the repo.

⚠️ **CRITICAL: Use content-level copy to avoid directory nesting.**

```powershell
$skillsSource = Join-Path $repoDir "skills"
$skillsTarget = Join-Path $copilotDir "skills"

$newSkills = @(); $updatedSkills = @(); $removedSkills = @(); $unchangedCount = 0
$repoSkillNames = @()

# Add new and update existing
Get-ChildItem $skillsSource -Directory | ForEach-Object {
    $skillName = $_.Name
    $repoSkillNames += $skillName
    $targetDir = Join-Path $skillsTarget $skillName

    if (-not (Test-Path $targetDir)) {
        # NEW — create directory and copy contents INTO it
        New-Item -ItemType Directory -Path $targetDir -Force | Out-Null
        Copy-Item "$($_.FullName)\*" $targetDir -Recurse -Force
        $newSkills += $skillName
    } else {
        # EXISTING — compare SKILL.md hashes
        $repoFile = Join-Path $_.FullName "SKILL.md"
        $localFile = Join-Path $targetDir "SKILL.md"
        if ((Test-Path $repoFile) -and (Test-Path $localFile)) {
            if ((Get-FileHash $repoFile -Algorithm MD5).Hash -ne (Get-FileHash $localFile -Algorithm MD5).Hash) {
                Copy-Item "$($_.FullName)\*" $targetDir -Recurse -Force
                $updatedSkills += $skillName
            } else {
                $unchangedCount++
            }
        }
    }
}

# Remove local skills that no longer exist in GitHub
Get-ChildItem $skillsTarget -Directory | ForEach-Object {
    if ($_.Name -notin $repoSkillNames) {
        Remove-Item $_.FullName -Recurse -Force
        $removedSkills += $_.Name
    }
}
```

> **Why `Copy-Item "$($_.FullName)\*"` instead of `Copy-Item $_.FullName`?**
> `Copy-Item <folder> <existing-folder> -Recurse` creates a nested subfolder: `skills/close-day/close-day/SKILL.md`.
> `Copy-Item <folder>\* <existing-folder> -Recurse` copies the CONTENTS into the target — correct behavior.

### Step 4: Sync Instructions (Full Mirror)
```powershell
$instrSource = Join-Path $repoDir "instructions"
$instrTarget = Join-Path $copilotDir "instructions"

$newInstr = @(); $updInstr = @(); $removedInstr = @()
$repoInstrNames = @()

Get-ChildItem $instrSource -Filter "*.md" | ForEach-Object {
    $repoInstrNames += $_.Name
    $target = Join-Path $instrTarget $_.Name
    if (-not (Test-Path $target)) {
        Copy-Item $_.FullName $target -Force
        $newInstr += $_.Name
    } elseif ((Get-FileHash $_.FullName -Algorithm MD5).Hash -ne (Get-FileHash $target -Algorithm MD5).Hash) {
        Copy-Item $_.FullName $target -Force
        $updInstr += $_.Name
    }
}

# Remove local instructions not in GitHub
Get-ChildItem $instrTarget -Filter "*.md" | Where-Object { $_.Name -notin $repoInstrNames } | ForEach-Object {
    Remove-Item $_.FullName -Force
    $removedInstr += $_.Name
}
```

### Step 5: Sync Agents (Full Mirror)
```powershell
$agentsSource = Join-Path $repoDir "agents"
$agentsTarget = Join-Path $copilotDir "agents"

$newAgents = @(); $updAgents = @(); $removedAgents = @()
$repoAgentNames = @()

Get-ChildItem $agentsSource -Filter "*.md" -ErrorAction SilentlyContinue | ForEach-Object {
    $repoAgentNames += $_.Name
    $target = Join-Path $agentsTarget $_.Name
    if (-not (Test-Path $target)) {
        Copy-Item $_.FullName $target -Force
        $newAgents += $_.Name
    } elseif ((Get-FileHash $_.FullName -Algorithm MD5).Hash -ne (Get-FileHash $target -Algorithm MD5).Hash) {
        Copy-Item $_.FullName $target -Force
        $updAgents += $_.Name
    }
}

# Remove local agents not in GitHub
Get-ChildItem $agentsTarget -Filter "*.md" -ErrorAction SilentlyContinue | Where-Object { $_.Name -notin $repoAgentNames } | ForEach-Object {
    Remove-Item $_.FullName -Force
    $removedAgents += $_.Name
}
```

### Step 6: Report Results
```markdown
## ✅ Skills Synced from GitHub

| Category | Total | New | Updated | Removed | Unchanged |
|----------|-------|-----|---------|---------|-----------|
| Skills | N | N | N | N | N |
| Instructions | N | N | N | N | N |
| Agents | N | N | N | N | N |

### New
- + skill-name

### Updated
- ~ modified-skill

### Removed
- - deleted-skill

📂 Installed to: ~/.copilot/
🔗 Source: https://github.com/piusn/skills-plugin-marketplace
```

If nothing changed, report: "✅ Already up to date — local .copilot/ matches GitHub."

## Error Handling
- **Repository not found:** Search known locations, then ask the user for the path. Clone as last resort.
- **Pull fails:** Warn the user. Suggest `git stash` or `git reset --hard origin/main`.
- **Source directory missing in repo:** Skip that category with a warning.
- **No .copilot directory:** Create it — this is the new machine scenario.

## Key Design Decisions

### Full mirror (GitHub is source of truth)
Local `.copilot/` is made to exactly match GitHub. Items removed from the repo are removed locally. If you have local-only WIP skills, push them to GitHub first using `sync-skills-to-github`.

### Content-level copy (not folder copy)
Always use `Copy-Item "$source\*" $target` instead of `Copy-Item $source $target` when the target folder already exists. This prevents the PowerShell nesting bug where `Copy-Item <folder> <existing-folder> -Recurse` creates `target/foldername/foldername/`.

### Hash-based change detection
MD5 hash comparison on `SKILL.md` detects actual content changes, providing accurate reporting of new vs updated vs unchanged.

## Tools & APIs Used
- `powershell` — File operations and git commands
- `git` — Version control operations

## Notes
- This skill is **read-only on the repo** — it only modifies `~/.copilot/`, never commits or pushes
- Always use `$env:USERPROFILE` or `$HOME` — never hardcode a username
- Zip files and temporary artifacts in the skills directory are NOT synced
- On a brand new machine, all directories are created automatically
