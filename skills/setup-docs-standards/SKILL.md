---
description: "Set up documentation standards, folder structure, and contributing guides for any solution. Use this skill when the user says 'setup docs standards', 'documentation structure', 'scaffold docs', 'setup contributing guides', 'project documentation setup', 'docs standards', or 'initialize documentation'. Creates CONTRIBUTING.md files, docs folder structure (bugs, features, designs, user-guides, setup-guides), documentation standards instruction files, and work type standards for new or existing solutions."
---

# Documentation Standards Setup

Set up a comprehensive documentation structure and contributing standards for any solution — new or existing. This skill creates the complete scaffolding, standards files, and project-level contributing guides.

## When to Use

- Setting up a new repository/solution
- Adding documentation standards to an existing solution that lacks them
- Standardizing documentation across a multi-project solution
- When asked to "set up docs", "scaffold documentation", or "initialize contributing guides"

## What Gets Created

### Solution-Level Files

| File | Purpose |
|------|---------|
| `CONTRIBUTING.md` | Root entry point for contributors — links to all standards |
| `.github/instructions/documentation-standards.instructions.md` | Documentation hierarchy, folder structure, naming conventions, staleness rules |
| `.github/instructions/work-type-standards.instructions.md` | Requirements per work type (Feature, Bug Fix, Change Request, Refactoring, Testing) |
| `docs/DOCS-INDEX.md` | Solution-level documentation index |

### Per-Project Files

For every project (identified by `.csproj`, `package.json`, or similar project markers):

| File/Folder | Purpose |
|-------------|---------|
| `CONTRIBUTING.md` | Project-specific contributing guide |
| `docs/bugs/.gitkeep` | Bug fix documentation folder |
| `docs/features/.gitkeep` | Feature documentation folder |
| `docs/designs/.gitkeep` | Design documentation folder |
| `docs/user-guides/.gitkeep` | User guides (how to use the tooling) |
| `docs/setup-guides/.gitkeep` | Setup guides (how to install/configure) |

## Execution Steps

### Step 1: Discover the solution structure

Identify all projects in the solution:

```powershell
# Find all project files
$cwd = Get-Location
$csproj = Get-ChildItem -Recurse -Filter "*.csproj" | Where-Object { $_.FullName -notmatch 'node_modules|\\\.git\\|\\bin\\|\\obj\\' }
$packageJson = Get-ChildItem -Recurse -Filter "package.json" -Depth 2 | Where-Object { $_.FullName -notmatch 'node_modules|\\\.git\\' }
$pyProject = Get-ChildItem -Recurse -Filter "pyproject.toml" | Where-Object { $_.FullName -notmatch 'node_modules|\\\.git\\|\\venv\\' }
$goMod = Get-ChildItem -Recurse -Filter "go.mod" | Where-Object { $_.FullName -notmatch 'vendor|\\\.git\\' }

Write-Host "Found: $($csproj.Count) C# projects, $($packageJson.Count) JS/TS projects, $($pyProject.Count) Python projects, $($goMod.Count) Go modules"
```

**IMPORTANT:** Present the discovered project list to the user (or reason about it) before proceeding. Exclude test output directories (`bin/`, `obj/`, `node_modules/`, `dist/`, `vendor/`).

### Step 2: Create solution-level CONTRIBUTING.md

Create `/CONTRIBUTING.md` at the repo root. Customize the content based on:
- The solution's tech stack (discovered from project files)
- Build commands (look for `Makefile`, `package.json` scripts, `.sln` files)
- The solution name (from repo directory name or README title)

**Template:**

```markdown
# Contributing to {SOLUTION_NAME}

Welcome! This document is the entry point for contributing to {SOLUTION_NAME}. It links to the detailed standards that govern how we write code, document changes, and maintain quality.

## Quick Links

| Standard | Description | Location |
|----------|-------------|----------|
| [Documentation Standards](.github/instructions/documentation-standards.instructions.md) | Where docs live, folder structure, naming conventions | `.github/instructions/` |
| [Work Type Standards](.github/instructions/work-type-standards.instructions.md) | Requirements per work type (feature, bug fix, refactor, etc.) | `.github/instructions/` |

## Contribution Workflow

### 1. Choose the Right Work Type

Every change falls into a work type. Each has specific requirements:

- **New Feature** — requires design doc, full tests, documentation, code review
- **Bug Fix** — requires regression test, doc update if affected
- **Change Request** — requires design doc if scope changes, update existing tests/docs
- **Refactoring** — existing tests must pass, document rationale if non-trivial
- **Testing** — test code is the deliverable, document test coverage additions

See [Work Type Standards](.github/instructions/work-type-standards.instructions.md) for the complete matrix.

### 2. Follow Documentation Standards

Documentation lives **close to where it's needed**:

- **Solution-level** (`/docs/`) — architecture, deployment, cross-cutting concerns
- **Project-level** (`{project}/docs/`) — project-specific guides, features, bugs, designs

See [Documentation Standards](.github/instructions/documentation-standards.instructions.md) for folder structure and naming conventions.

### 3. Build & Test Before Pushing

{INSERT BUILD COMMANDS BASED ON DISCOVERED TECH STACK}

## Project Structure

{INSERT DISCOVERED PROJECT TREE}
```

### Step 3: Create documentation-standards.instructions.md

Create `/.github/instructions/documentation-standards.instructions.md` with the `applyTo` front matter and comprehensive documentation hierarchy rules.

**The file must include these sections:**
1. Documentation Hierarchy (solution-level vs project-level)
2. Folder Structure (standard project layout + solution layout)
3. Naming Conventions (kebab-case files, lowercase folders)
4. README Requirements (title, purpose, quick start, structure, dependencies)
5. When to Create Documentation (trigger table)
6. When to Update Documentation (do/don't rules)
7. Staleness Rules
8. Cross-Referencing rules

**Front matter:**
```yaml
---
applyTo: "**/*.md,**/docs/**,**/CONTRIBUTING.md"
---
```

**Project-level required folders:**
```
docs/
├── bugs/                    # Bug fix documentation
├── features/                # Feature documentation
├── designs/                 # Design documents
├── user-guides/             # User guides (how to use)
└── setup-guides/            # Setup guides (how to install/configure)
```

### Step 4: Create work-type-standards.instructions.md

Create `/.github/instructions/work-type-standards.instructions.md` defining requirements per work type.

**Front matter:**
```yaml
---
applyTo: "**/*.cs,**/*.ts,**/*.tsx,**/*.md,**/*.py,**/*.go,**/*.java"
---
```

Adjust the `applyTo` pattern based on the solution's tech stack.

**Must include these work types with full process, required artifacts, and PR templates:**
- New Feature
- Bug Fix
- Feature Improvement
- Change Request
- Refactoring
- Testing

**Each work type section covers:**
- Required documentation artifacts (what, where)
- Required code artifacts (tests, migrations)
- Git branch naming convention
- PR description template
- Documentation update checklist

### Step 5: Create docs/DOCS-INDEX.md

Create a solution-level docs index. If the solution already has docs, catalog them by category. If not, create an empty index with the standard category sections:

```markdown
# Solution Documentation Index

## Architecture
{list or "No documents yet"}

## Feature Design
{list or "No documents yet"}

## Technical Documentation
{list or "No documents yet"}

## Analysis
{list or "No documents yet"}

## Deployment
{list or "No documents yet"}

## User Guides
{list or "No documents yet"}
```

### Step 6: Scaffold all projects

For each discovered project, create the standard documentation structure.

**Use PowerShell for efficiency — batch multiple projects per command:**

```powershell
$projects = @(
    # Discovered projects with their paths, names, and descriptions
    @{ Path = "src/MyProject"; Name = "My Project"; Desc = "Does X" }
    # ... more projects
)

foreach ($proj in $projects) {
    $projPath = Join-Path $cwd $proj.Path
    
    # Create docs subdirectories with .gitkeep
    @("docs/bugs", "docs/features", "docs/designs", "docs/user-guides", "docs/setup-guides") | ForEach-Object {
        $dir = Join-Path $projPath $_
        if (-not (Test-Path $dir)) {
            New-Item -ItemType Directory -Path $dir -Force | Out-Null
            New-Item -ItemType File -Path (Join-Path $dir ".gitkeep") -Force | Out-Null
        }
    }
    
    # Create CONTRIBUTING.md (skip if exists)
    $contributing = Join-Path $projPath "CONTRIBUTING.md"
    if (-not (Test-Path $contributing)) {
        # Calculate relative path back to repo root for links
        $depth = ($proj.Path -split '[/\\]').Count
        $relRoot = ("../" * $depth).TrimEnd('/')
        
        # Write CONTRIBUTING.md from template (customize per project)
        # ... (use the template from the standard)
    }
}
```

**CONTRIBUTING.md Template for Projects:**

```markdown
# Contributing to {PROJECT_NAME}

{PROJECT_DESCRIPTION}

## Standards

This project follows the solution-level standards defined in:
- [Solution Contributing Guide]({REL_PATH}/CONTRIBUTING.md)
- [Documentation Standards]({REL_PATH}/.github/instructions/documentation-standards.instructions.md)
- [Work Type Standards]({REL_PATH}/.github/instructions/work-type-standards.instructions.md)

## Project Documentation

| Folder | Purpose |
|--------|---------|
| `docs/bugs/` | Bug fix documentation — root cause analysis, regression notes |
| `docs/features/` | Feature documentation — design, implementation notes |
| `docs/designs/` | Design documents — proposals, architecture decisions |
| `docs/user-guides/` | User guides — how to use the tooling, workflows, tutorials |
| `docs/setup-guides/` | Setup guides — how to install, configure, and set up the tooling |

## Work Type Quick Reference

| Work Type | Required Docs |
|-----------|--------------|
| **New Feature** | Design doc in `docs/features/`, update this CONTRIBUTING.md |
| **Bug Fix** | Bug doc in `docs/bugs/{issue-id}-{desc}.md` with root cause |
| **Improvement** | Update existing feature doc in `docs/features/` |
| **Refactoring** | No doc required unless API changes |
```

### Step 7: Handle existing documentation

If the solution already has documentation scattered in non-standard locations:

1. **Audit** — list all `.md` files in the repo
2. **Classify** — categorize each as feature-design, bug, analysis, reference, etc.
3. **Propose moves** — present a move plan to the user (or decide autonomously)
4. **Execute** — use `git mv` to preserve history
5. **Update indexes** — update DOCS-INDEX.md and any README.md files with new paths

### Step 8: Verify

```powershell
# Count created files
$contribs = (Get-ChildItem -Recurse -Filter "CONTRIBUTING.md" | Where-Object { $_.FullName -notmatch 'node_modules' }).Count
$gitkeeps = (Get-ChildItem -Recurse -Filter ".gitkeep" | Where-Object { $_.FullName -notmatch 'node_modules' }).Count
Write-Host "CONTRIBUTING.md files: $contribs"
Write-Host ".gitkeep files: $gitkeeps"

# Verify key files exist
@("CONTRIBUTING.md", ".github/instructions/documentation-standards.instructions.md", ".github/instructions/work-type-standards.instructions.md") | ForEach-Object {
    if (Test-Path $_) { Write-Host "OK $_" } else { Write-Host "MISSING $_" }
}
```

## Customization Points

When applying to a new solution, adapt these based on context:

| Element | How to Customize |
|---------|-----------------|
| Tech stack in `applyTo` patterns | Match file extensions to the solution's languages |
| Build commands in CONTRIBUTING.md | Discover from Makefile, package.json, .sln, etc. |
| Branch naming convention | Match team's existing convention or use `user/{name}/{type}/{desc}` |
| Code review process | Adjust review model list to available models |
| Project descriptions | Infer from README, project file, or directory name |
| Relative link paths | Calculate based on directory depth from repo root |

## Notes

- `.gitkeep` files are empty — they exist only to make git track empty directories
- Never overwrite existing `CONTRIBUTING.md` files — they may have project-specific content
- The skill is idempotent — running it twice won't duplicate content
- Use `git mv` for any file moves to preserve git history
- The documentation standards instruction file uses `applyTo` for automatic Copilot integration
