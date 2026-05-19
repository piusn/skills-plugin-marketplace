---
description: "Set up documentation standards, folder structure, and contributing guides for any solution. Use this skill when the user says 'setup docs standards', 'documentation structure', 'scaffold docs', 'setup contributing guides', 'project documentation setup', 'docs standards', or 'initialize documentation'. Creates CONTRIBUTING.md files, docs folder structure (bugs, features, user-guides, tasks, refactoring, change-requests, decisions), documentation standards instruction files, versioned file naming, changelog tracking, and work type standards for new or existing solutions."
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
| `docs/bugs/README.md` | Bug fix documentation — root cause analysis, regression notes |
| `docs/features/README.md` | Feature documentation — design, implementation notes |
| `docs/user-guides/README.md` | User guides — how to use the tooling, workflows, tutorials |
| `docs/tasks/README.md` | Task documentation — upgrades, migrations, one-off work items (**not** the kanban board; see note below) |
| `docs/refactoring/README.md` | Refactoring documentation — rationale, scope, impact |
| `docs/change-requests/README.md` | Change request documentation — scope changes, approvals |
| `docs/decisions/README.md` | Architecture Decision Records (ADRs) — why we chose X over Y |
| `docs/plans/README.md` | Plans — implementation plans, research plans, design proposals, system designs |

### Folder Structure

```
{project}/docs/
├── bugs/              ← v1.2-AB12345-cab-download-timeout.md
├── features/          ← v1.3-local-disk-cab-support.md
├── user-guides/       ← local-setup.md, creating-test-runs.md
├── tasks/             ← v1.2-upgrade-to-net8.md
├── refactoring/       ← v1.1-consolidate-ef-data-layer.md
├── change-requests/   ← v1.3-CR-add-parent-package-field.md
├── decisions/         ← ADR-0001-use-shared-ef-library.md
└── plans/             ← implementation-plan-user-dashboard.md, research-api-gateway.md
```

> **Heads up — `docs/tasks/` is not the kanban board.** This folder holds *task documentation* (versioned write-ups for upgrades, migrations, one-off work items). The active kanban board lives in **Daily Planner** (issue #37 retired the file-board snapshot at `C:\boards\`). See the **start-task** skill's [MCP-ONLY MODE](../start-task/SKILL.md) banner for the canonical flow.

There are two levels of `plans/` directories:

**Root-level plans** (`/docs/plans/`):
- Cross-cutting plans that span multiple projects (e.g., system designs, capacity plans, design proposals, migration roadmaps)
- Named: `{plan-type}-{description}.md` (e.g., `system-design-event-pipeline.md`, `capacity-plan-q3-scaling.md`)

**Project-level plans** (`{project}/docs/plans/`):
- Plans scoped to a single project (e.g., implementation plans, research plans, refactoring plans)
- Named: `{plan-type}-{description}.md` (e.g., `implementation-plan-user-dashboard.md`, `research-caching-strategy.md`)

**Plan type prefixes:**

| Prefix | Use Case | Example |
|--------|----------|---------|
| `implementation-plan-` | Engineering task implementation plans | `implementation-plan-auth-module.md` |
| `research-` | Research plans and findings | `research-api-gateway-options.md` |
| `system-design-` | System design documents | `system-design-event-pipeline.md` |
| `design-proposal-` | Design proposals with migration roadmaps | `design-proposal-microservices-migration.md` |
| `capacity-plan-` | Capacity planning documents | `capacity-plan-q3-scaling.md` |

**Rule:** If a plan spans multiple projects or is solution-wide, it goes in `/docs/plans/`. If it's scoped to a single project, it goes in `{project}/docs/plans/`.

### File Naming & Versioning

All versioned documents use the pattern: `v{major}.{minor}-{short-description}.md`

| Folder | Naming Pattern | Example |
|--------|---------------|---------|
| `bugs/` | `v{version}-{ticket-id}-{description}.md` | `v1.2-AB12345-cab-download-timeout.md` |
| `features/` | `v{version}-{description}.md` | `v1.3-local-disk-cab-support.md` |
| `user-guides/` | `{description}.md` (no version prefix) | `local-setup.md`, `creating-test-runs.md` |
| `tasks/` | `v{version}-{description}.md` | `v1.2-upgrade-to-net8.md` |
| `refactoring/` | `v{version}-{description}.md` | `v1.1-consolidate-ef-data-layer.md` |
| `change-requests/` | `v{version}-CR-{description}.md` | `v1.3-CR-add-parent-package-field.md` |
| `decisions/` | `ADR-{nnnn}-{description}.md` | `ADR-0001-use-shared-ef-library.md` |
| `plans/` | `{plan-type}-{description}.md` | `implementation-plan-user-dashboard.md` |

### Changelog Tracking

Every versioned document **must** include a changelog section at the bottom. When updating an existing document, append a new entry — never overwrite history.

**Template for new documents:**

```markdown
## Changelog

| Date | Author | Change |
|------|--------|--------|
| 2026-04-30 | @username | Initial version |
```

**When updating an existing document**, add a new row at the top of the changelog table:

```markdown
## Changelog

| Date | Author | Change |
|------|--------|--------|
| 2026-05-15 | @jane | Updated steps for .NET 9 migration path |
| 2026-04-30 | @username | Initial version |
```

**Rules:**
- Newest entry goes at the **top** of the table (reverse chronological)
- `Author` uses the Git/GitHub username prefixed with `@`
- `Change` is a concise summary of what was modified
- Never delete or modify existing changelog entries
- If a document is superseded, add a final changelog entry linking to the replacement

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
├── bugs/                    # Bug fix documentation (versioned)
├── features/                # Feature documentation (versioned)
├── user-guides/             # User guides (how to use)
├── tasks/                   # Task documentation — upgrades, migrations (versioned)
├── refactoring/             # Refactoring documentation — rationale, scope (versioned)
├── change-requests/         # Change request documentation (versioned)
├── decisions/               # Architecture Decision Records (ADRs)
└── plans/                   # Plans — implementation, research, design, capacity (prefixed by type)
```

Each folder contains a `README.md` that describes the folder's purpose and lists its contents. This ensures the folder is always committed to git (no need for `.gitkeep`).

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

## Plans
{list or "No documents yet — implementation plans, research plans, design proposals, and system designs go here"}

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

# Standard folder definitions with README content
$docFolders = @(
    @{ Name = "bugs"; Title = "Bug Documentation"; Desc = "Root cause analyses, regression notes, and bug fix documentation.`nFiles are named: ``v{version}-{ticket-id}-{description}.md``" }
    @{ Name = "features"; Title = "Feature Documentation"; Desc = "Feature design documents, implementation notes, and specifications.`nFiles are named: ``v{version}-{description}.md``" }
    @{ Name = "user-guides"; Title = "User Guides"; Desc = "How-to guides, workflows, and tutorials for using this project.`nFiles are named: ``{description}.md`` (no version prefix)" }
    @{ Name = "tasks"; Title = "Task Documentation"; Desc = "Task documentation for upgrades, migrations, and one-off work items.`nFiles are named: ``v{version}-{description}.md``" }
    @{ Name = "refactoring"; Title = "Refactoring Documentation"; Desc = "Refactoring rationale, scope, and impact documentation.`nFiles are named: ``v{version}-{description}.md``" }
    @{ Name = "change-requests"; Title = "Change Requests"; Desc = "Change request documentation — scope changes, approvals, and impact.`nFiles are named: ``v{version}-CR-{description}.md``" }
    @{ Name = "decisions"; Title = "Architecture Decision Records"; Desc = "ADRs documenting why we chose X over Y.`nFiles are named: ``ADR-{nnnn}-{description}.md``" }
    @{ Name = "plans"; Title = "Plans"; Desc = "Implementation plans, research plans, design proposals, system designs, and capacity plans.`nFiles are named: ``{plan-type}-{description}.md`` (e.g., ``implementation-plan-auth-module.md``, ``research-caching-strategy.md``)" }
)

foreach ($proj in $projects) {
    $projPath = Join-Path $cwd $proj.Path
    
    # Create docs subdirectories with README.md
    foreach ($folder in $docFolders) {
        $dir = Join-Path $projPath "docs/$($folder.Name)"
        if (-not (Test-Path $dir)) {
            New-Item -ItemType Directory -Path $dir -Force | Out-Null
        }
        $readme = Join-Path $dir "README.md"
        if (-not (Test-Path $readme)) {
            $content = "# $($folder.Title)`n`n$($folder.Desc)`n`n## Documents`n`n_No documents yet._`n"
            Set-Content -Path $readme -Value $content -Encoding utf8NoBOM
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

| Folder | Purpose | Naming |
|--------|---------|--------|
| `docs/bugs/` | Bug fix documentation — root cause analysis, regression notes | `v{version}-{ticket}-{desc}.md` |
| `docs/features/` | Feature documentation — design, implementation notes | `v{version}-{desc}.md` |
| `docs/user-guides/` | User guides — how to use the tooling, workflows, tutorials | `{desc}.md` |
| `docs/tasks/` | Task documentation — upgrades, migrations, one-off work | `v{version}-{desc}.md` |
| `docs/refactoring/` | Refactoring documentation — rationale, scope, impact | `v{version}-{desc}.md` |
| `docs/change-requests/` | Change request documentation — scope changes, approvals | `v{version}-CR-{desc}.md` |
| `docs/decisions/` | Architecture Decision Records (ADRs) | `ADR-{nnnn}-{desc}.md` |
| `docs/plans/` | Plans — implementation, research, design proposals, system designs | `{plan-type}-{desc}.md` |

## Versioning & Changelogs

- Versioned files use the prefix `v{major}.{minor}-` (e.g., `v1.2-upgrade-to-net8.md`)
- Every versioned document includes a **Changelog** section at the bottom
- When updating an existing document, add a new changelog entry (newest first) — never overwrite history
- Changelog entries include: **Date**, **Author** (`@username`), and **Change** summary

## Work Type Quick Reference

| Work Type | Required Docs |
|-----------|--------------|
| **New Feature** | Design doc in `docs/features/`, update this CONTRIBUTING.md |
| **Bug Fix** | Bug doc in `docs/bugs/v{ver}-{issue-id}-{desc}.md` with root cause |
| **Improvement** | Update existing feature doc in `docs/features/` |
| **Task** | Task doc in `docs/tasks/v{ver}-{desc}.md` |
| **Refactoring** | Refactoring doc in `docs/refactoring/v{ver}-{desc}.md` if non-trivial |
| **Change Request** | CR doc in `docs/change-requests/v{ver}-CR-{desc}.md` |
| **Decision** | ADR in `docs/decisions/ADR-{nnnn}-{desc}.md` |
| **Implementation Plan** | Plan in `docs/plans/implementation-plan-{desc}.md` |
| **Research** | Plan in `docs/plans/research-{desc}.md` |
| **System Design** | Plan in `docs/plans/system-design-{desc}.md` |
| **Design Proposal** | Plan in `docs/plans/design-proposal-{desc}.md` |
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
$readmes = (Get-ChildItem -Recurse -Path "*/docs/*" -Filter "README.md" | Where-Object { $_.FullName -notmatch 'node_modules' }).Count
Write-Host "CONTRIBUTING.md files: $contribs"
Write-Host "docs/*/README.md files: $readmes"

# Verify key files exist
@("CONTRIBUTING.md", ".github/instructions/documentation-standards.instructions.md", ".github/instructions/work-type-standards.instructions.md") | ForEach-Object {
    if (Test-Path $_) { Write-Host "OK $_" } else { Write-Host "MISSING $_" }
}

# Verify all 8 doc folders have README.md per project
$docFolders = @("bugs", "features", "user-guides", "tasks", "refactoring", "change-requests", "decisions", "plans")
# ... check each project's docs/ subfolders
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

- Each `docs/*/README.md` describes the folder's purpose, naming convention, and lists its contents — this replaces `.gitkeep` and ensures folders are always committed
- Never overwrite existing `CONTRIBUTING.md` files — they may have project-specific content
- The skill is idempotent — running it twice won't duplicate content
- Use `git mv` for any file moves to preserve git history
- The documentation standards instruction file uses `applyTo` for automatic Copilot integration
- All versioned documents must include a **Changelog** table at the bottom (date, author, change summary)
- When updating existing docs, always append to the changelog — never remove or modify previous entries
- Changelog entries are reverse chronological (newest first)
