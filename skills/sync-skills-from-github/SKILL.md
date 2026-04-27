---
description: "Sync skills, instructions, and agents from the skills-plugin-marketplace GitHub repository to the local .copilot directory. Use this skill when the user says 'sync skills from github', 'pull skills', 'download skills', 'restore skills', 'sync from repo', 'install skills', or 'set up skills on new machine'. Pulls the latest from GitHub and copies new/updated files into ~/.copilot/. This is the PULL direction — it brings remote changes into the local environment."
---

# Sync Skills from GitHub

## Purpose
**Direction: GitHub → Local `.copilot/`**

Pull the latest skills, instructions, and agents from the `skills-plugin-marketplace` GitHub repository into the local `~/.copilot/` directory. This adds new items and updates existing ones — it does NOT delete local items that don't exist in the repo (the user may have work-in-progress skills locally).

> **Counterpart skill:** `sync-skills-to-github` does the reverse — pushes local `.copilot/` content TO the repository.

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

### Step 3: Sync Skills (Additive — No Deletions)
Copy skill directories from repo to local. **Add new skills and update existing ones. Do NOT delete local-only skills.**

⚠️ **CRITICAL: Use content-level copy to avoid directory nesting.**

```powershell
$skillsSource = Join-Path $repoDir "skills"
$skillsTarget = Join-Path $copilotDir "skills"

$newSkills = @(); $updatedSkills = @(); $unchangedCount = 0

Get-ChildItem $skillsSource -Directory | ForEach-Object {
    $skillName = $_.Name
    $targetDir = Join-Path $skillsTarget $skillName

    if (-not (Test-Path $targetDir)) {
        # NEW skill — create directory and copy contents INTO it
        New-Item -ItemType Directory -Path $targetDir -Force | Out-Null
        Copy-Item "$($_.FullName)\*" $targetDir -Recurse -Force
        $newSkills += $skillName
    } else {
        # EXISTING — compare SKILL.md hashes to detect changes
        $repoFile = Join-Path $_.FullName "SKILL.md"
        $localFile = Join-Path $targetDir "SKILL.md"
        if ((Test-Path $repoFile) -and (Test-Path $localFile)) {
            $repoHash = (Get-FileHash $repoFile -Algorithm MD5).Hash
            $localHash = (Get-FileHash $localFile -Algorithm MD5).Hash
            if ($repoHash -ne $localHash) {
                # UPDATED — overwrite contents
                Copy-Item "$($_.FullName)\*" $targetDir -Recurse -Force
                $updatedSkills += $skillName
            } else {
                $unchangedCount++
            }
        }
    }
}
```

> **Why `Copy-Item "$($_.FullName)\*"` instead of `Copy-Item $_.FullName`?**
> Using `Copy-Item <folder> <existing-folder> -Recurse` creates a nested subfolder (e.g., `skills/close-day/close-day/SKILL.md`).
> Using `Copy-Item <folder>\* <existing-folder> -Recurse` copies the CONTENTS into the target — which is what we want.

### Step 4: Sync Instructions (Additive)
```powershell
$instrSource = Join-Path $repoDir "instructions"
$instrTarget = Join-Path $copilotDir "instructions"

$newInstr = @(); $updInstr = @()
Get-ChildItem $instrSource -Filter "*.md" | ForEach-Object {
    $target = Join-Path $instrTarget $_.Name
    if (-not (Test-Path $target)) {
        Copy-Item $_.FullName $target -Force
        $newInstr += $_.Name
    } elseif ((Get-FileHash $_.FullName -Algorithm MD5).Hash -ne (Get-FileHash $target -Algorithm MD5).Hash) {
        Copy-Item $_.FullName $target -Force
        $updInstr += $_.Name
    }
}
```

### Step 5: Sync Agents (Additive)
```powershell
$agentsSource = Join-Path $repoDir "agents"
$agentsTarget = Join-Path $copilotDir "agents"

$newAgents = @(); $updAgents = @()
Get-ChildItem $agentsSource -Filter "*.md" -ErrorAction SilentlyContinue | ForEach-Object {
    $target = Join-Path $agentsTarget $_.Name
    if (-not (Test-Path $target)) {
        Copy-Item $_.FullName $target -Force
        $newAgents += $_.Name
    } elseif ((Get-FileHash $_.FullName -Algorithm MD5).Hash -ne (Get-FileHash $target -Algorithm MD5).Hash) {
        Copy-Item $_.FullName $target -Force
        $updAgents += $_.Name
    }
}
```

### Step 6: Report Results
Present a summary showing what changed:

```markdown
## ✅ Skills Synced from GitHub

| Category | Total | New | Updated | Unchanged |
|----------|-------|-----|---------|-----------|
| Skills | N | N | N | N |
| Instructions | N | N | N | N |
| Agents | N | N | N | N |

### New
- + skill-name
- + another-skill

### Updated
- ~ modified-skill
- ~ another-modified

📂 Installed to: ~/.copilot/
🔗 Source: https://github.com/piusn/skills-plugin-marketplace
```

If nothing changed, report: "✅ Already up to date — no new or updated files in the repository."

## Error Handling
- **Repository not found:** Search known locations, then ask the user for the path. Clone as last resort.
- **Pull fails:** Warn the user. Suggest `git stash` or `git reset --hard origin/main`.
- **Source directory missing in repo:** Skip that category with a warning.
- **No .copilot directory:** Create it — this is the new machine scenario.

## Tools & APIs Used
- `powershell` — File operations and git commands
- `git` — Version control operations

## Key Design Decisions

### Additive sync (no deletions)
Local-only skills are preserved. The user may have work-in-progress skills that haven't been pushed to the repo yet. Deleting them would cause data loss.

### Content-level copy (not folder copy)
Always use `Copy-Item "$source\*" $target` instead of `Copy-Item $source $target` when the target folder already exists. This prevents the PowerShell nesting bug where `Copy-Item <folder> <existing-folder> -Recurse` creates `target/foldername/foldername/`.

### Hash-based change detection
Use MD5 hash comparison on `SKILL.md` to detect actual content changes, rather than copying everything unconditionally. This provides accurate reporting of what was new vs updated vs unchanged.

## Notes
- This skill is **read-only** — it only modifies `~/.copilot/`, never the repository
- No git commits or pushes are made — this is purely a pull + copy operation
- Zip files and temporary artifacts in the skills directory are NOT synced
- Always use `$env:USERPROFILE` or `$HOME` — never hardcode a username in paths
- On a brand new machine, all directories are created automatically
