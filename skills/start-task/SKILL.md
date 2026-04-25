---
name: start-task
description: >
  Start working on a Daily Planner task with a dedicated workspace, session, and
  workflow. Use this skill when the user says "start task", "work on task",
  "begin task", or references starting any task by ID. Sets up the workspace,
  detects the task type, and delegates to the appropriate workflow skill.
---

# Start Task — Orchestrator Skill

Sets up a dedicated workspace and session for each Daily Planner task, then delegates to the appropriate workflow based on task type.

**⛔ Critical Rule: One task per session.** Every task gets its own Copilot session. Never share a session between tasks.

## Instructions

When the user wants to start working on a task, follow these steps in order:

### Step 0: Session Isolation Check

Before anything else, evaluate whether this task belongs in the current session:

1. **Check if a task is already active in this session:**
   - Look at the current working directory — is it already inside a task workspace (`C:\repositories\personal-copilot-agent\*`)?
   - Was a workflow already invoked in this session?

2. **If a task is already active:**
   - The new task **must** start in a new session. Prompt the user:
     ```
     ⚠️ This session is already working on: {current task title}
     
     Each task requires its own session. Please:
     1. Run /exit to close this session
     2. Open a new terminal
     3. Run: copilot
     4. Run: /rename {suggested session name}
     5. Then say: "start task {task id}"
     ```
   - **Do NOT proceed** with the new task in the current session.

3. **If no task is active:** Proceed to Step 1.

### Step 1: Ensure task exists in Daily Planner

#### If the user provides a task ID:
1. Call `DailyPlanner-get_task` with the provided task ID to get full details (title, description, tags, priority, etc.)
2. Also call `DailyPlanner-get_task_prompt` with the task ID to get the AI prompt (if available)

#### If the user describes a task without an ID:
1. Search for it: `DailyPlanner-search_tasks(query: "[task description]")`
2. **If found:** Confirm with the user and use the matched task
3. **If not found:** Create it in Daily Planner first:
   - Ask the user for: title, description, priority, and tags
   - Call `DailyPlanner-create_task` with the details
   - Use the returned task ID going forward

**Every task must exist in Daily Planner before work begins.**

### Step 1b: Session Naming

Name the session using the format: `{Integer Task ID}-{Task Title}`

Examples:
- Task #142 "Build Email Template" → session name: `142-Build Email Template`
- Task #87 "Fix Login Timeout" → session name: `87-Fix Login Timeout`

If already in the correct session, rename it:
```
/rename {Integer Task ID}-{Task Title}
```

If the user needs to start a new session, include the rename command in the instructions.

### Step 2: Check for existing workspace (Resume Support)

Before creating a new workspace, check if one already exists for this task:

1. **Scan** `C:\repositories\personal-copilot-agent\` for a directory starting with the integer task ID (e.g., `142-*`)
2. **If found**, check for `progress.json` in the workspace:
   - If `progress.json` exists, read it and show the user:
     ```
     🔄 Existing workspace found: {directory-name}
     
     Last session: {timestamp}
     Workflow: {workflow_type}
     Current phase: {current_phase}
     Completed: {completed_phases}
     
     Resume from Phase {N}, or start fresh?
     ```
   - If the user chooses to resume: `cd` into the workspace and invoke the workflow skill, informing it to start at the recorded phase
   - If the user chooses fresh: Delete `progress.json` and proceed normally
3. **If no workspace found**, proceed to Step 3 (create workspace)

#### progress.json Format
Each workflow writes this file at phase transitions:
```json
{
  "taskId": "abc123",
  "workflow": "engineering-task",
  "currentPhase": "Phase 4: Implementation",
  "completedPhases": ["Phase 1: Planning", "Phase 2: Design", "Phase 3: Review"],
  "lastUpdated": "2026-03-19T11:00:00Z",
  "notes": "Design approved, starting implementation"
}
```

Workflows should update `progress.json` at the start of each new phase. This enables multi-day task continuity.

### Step 3: Start the task in Daily Planner

1. Call `DailyPlanner-start_task` with the task ID to set status to "In Progress"

### Step 4: Create the workspace directory

**Workspace root:** `C:\repositories\personal-copilot-agent\`

1. **Build the directory name** using the session name format: `{Integer Task ID}-{Task Title}`
   - Replace spaces with hyphens, remove special characters, lowercase everything
   - Example: Task #142 "Build Email Template" → `142-build-email-template`
   - Example: Task #87 "Fix Login Timeout" → `87-fix-login-timeout`
2. **Create the directory** at `C:\repositories\personal-copilot-agent\{directory-name}\`
3. **Create a README.md** inside the directory with the task context:

```markdown
# {Task Title}

**Task ID:** {taskId}
**Priority:** {priority}
**Type:** {type}
**Tags:** {tags}
**Due Date:** {dueDate or "None"}
**Workflow:** {detected workflow type}

## Description

{task description}

## AI Prompt

{AI prompt content if available, otherwise "No AI prompt configured for this task."}

## Documentation
All plans, designs, and documentation for this task live in `system-documentation/`.

## Definition of Done

{Generated from task details — see Step 4b below}
```

### Step 4b: Define the Definition of Done (DoD)

Every task must have a clear, measurable Definition of Done. Generate the initial DoD based on the task type, description, and workflow, then **ask the user to review and refine it**.

#### DoD Template
Add to the workspace README.md:
```markdown
## Definition of Done

### Acceptance Criteria
- [ ] [Specific, measurable criterion derived from task description]
- [ ] [Another criterion]
- [ ] [Another criterion]

### Quality Gates
- [ ] Code reviewed (multi-model or peer review as appropriate)
- [ ] Tests pass (unit, integration as applicable)
- [ ] No known bugs introduced
- [ ] Documentation updated (if applicable)

### Workflow-Specific Criteria
{Generated based on detected workflow — see table below}
```

#### Workflow-Specific DoD Defaults

| Workflow | Default DoD Criteria |
|----------|---------------------|
| Engineering | Design doc approved, all tests pass, code reviewed (multi-model), deployed to staging, validated |
| System Design | All phases documented, multi-model review passed, follow-up tasks created |
| System Docs | All components documented, diagrams validated against code, gap analysis complete |
| Design Proposal | Current state analyzed, gaps mapped, proposed design reviewed, migration roadmap defined |
| Quick Fix | Root cause identified, fix implemented, regression test added, tests pass |
| Research | Research questions answered, findings documented, follow-up tasks created |
| Documentation | Content accurate against codebase, reviewed for clarity, published to target location |
| Admin | All deliverables sent/published, stakeholders acknowledged, outcomes documented |

#### DoD Lifecycle
1. **At task start:** Generate initial DoD and ask user to review/refine
2. **During task:** If scope changes or new requirements emerge, **update the DoD** in README.md and inform the user
3. **Before completion:** Review each DoD item — all must be checked before marking the task done
4. **At completion:** Include DoD status in the completion summary

### Step 5: Detect task workflow type

Determine the appropriate workflow using this priority order:

#### 1. Tag-based detection (highest priority)
Check the task's tags for explicit workflow indicators:

| Tag | Workflow Skill |
|-----|---------------|
| `engineering`, `feature`, `build` | `engineering-task` |
| `system-design`, `architecture`, `design-system` | `workflow-system-design` |
| `system-docs`, `reverse-engineer`, `codebase-docs`, `architecture-docs` | `workflow-system-docs` |
| `design-proposal`, `redesign`, `migration`, `improvement-plan` | `workflow-design-proposal` |
| `quickfix`, `bugfix`, `hotfix`, `patch` | `workflow-quickfix` |
| `research`, `investigate`, `explore`, `spike` | `workflow-research` |
| `docs`, `documentation`, `guide`, `runbook` | `workflow-documentation` |
| `admin`, `coordination`, `process`, `communication` | `workflow-admin` |

#### 2. Keyword inference (if no matching tag)
Scan the task title and description for signals:

| Signal words | Inferred Workflow |
|-------------|------------------|
| "design", "implement", "build", "create feature" | `engineering-task` |
| "system design", "architect", "design a service", "scalability", "design system" | `workflow-system-design` |
| "document this system", "map this codebase", "reverse engineer", "system documentation", "architecture docs", "document existing" | `workflow-system-docs` |
| "propose design", "redesign", "improve architecture", "migration plan", "propose improvements", "design proposal" | `workflow-design-proposal` |
| "fix bug", "patch", "hotfix", "resolve issue", "broken" | `workflow-quickfix` |
| "investigate", "research", "analyze", "explore", "evaluate", "compare", "spike" | `workflow-research` |
| "document", "write guide", "update docs", "runbook", "onboarding" | `workflow-documentation` |
| "coordinate", "organize", "schedule", "draft email", "prepare", "review" | `workflow-admin` |

#### 3. Ask the user (fallback)
If the type can't be inferred, present choices:
> "What kind of work is this task? I'll use the right workflow:"
> 1. **Engineering** — full design → review → implement → deploy lifecycle
> 2. **System Design** — requirements → capacity → API → data model → architecture → trade-offs
> 3. **System Documentation** — reverse-engineer existing codebase → architecture diagrams → engineering audit
> 4. **Design Proposal** — analyze current system → identify gaps → propose improvements → migration roadmap
> 5. **Quick Fix** — streamlined branch → fix → test → PR
> 6. **Research** — investigate → synthesize → document findings
> 7. **Documentation** — analyze → write → review → publish
> 8. **Admin/Coordination** — plan → execute → follow-up → close

### Step 6: Start the workflow

After creating the directory, change to the workspace directory and start the detected workflow:

```
✅ Task workspace created at:
   C:\repositories\personal-copilot-agent\{directory-name}\

🔄 Detected workflow: {workflow type}
📋 Definition of Done: [summary of DoD criteria]

Starting the {workflow skill name} workflow...
```

**Default action:** `cd` into the workspace directory and invoke the detected workflow skill in the current session.

**Alternative:** If the user prefers a fresh session, guide them:
> "To work in a separate session instead: run /exit, then open a terminal in the workspace directory, run `copilot`, and invoke the workflow skill."

### Step 7: Check for "official" tag

If the task has an "official" or "Official" tag, inform the user:
> "This task is tagged as official. Consider invoking the **impact-tracker** skill to plan and document your impact for performance reviews."

### Step 8: Invoke the workflow

Invoke the appropriate workflow skill in the current session:
- `engineering-task` — for engineering tasks
- `workflow-system-design` — for system design tasks
- `workflow-system-docs` — for documenting existing systems
- `workflow-design-proposal` — for proposing designs/redesigns
- `workflow-quickfix` — for quick fixes
- `workflow-research` — for research tasks
- `workflow-documentation` — for documentation tasks
- `workflow-admin` — for admin/coordination tasks

## Handling Edge Cases

- **Task not found:** Inform the user and ask them to verify the task ID
- **Directory already exists:** Inform the user the workspace already exists and ask if they want to reuse it or create a new one with a suffix
- **Task already in progress:** Still create the directory if it doesn't exist, and inform the user the task was already in progress
- **No task ID provided:** Ask the user for the task ID, or suggest using `DailyPlanner-get_tasks` to find it
- **DailyPlanner unavailable:** Skip task tracking steps, warn the user, and proceed with workspace setup and workflow
- **Multiple matching tags:** Use the first match in the priority order above
- **User overrides detected type:** Always respect the user's choice over auto-detection

## Workflow Quick Reference

| Workflow | Best For | Key Phases | Skill Name |
|----------|----------|------------|------------|
| Engineering | Features, refactors, complex changes | Design → Multi-model review → Implement → Code review → Deploy | `engineering-task` |
| System Design | New services, architecture, scaling | Requirements → Capacity → API → Data Model → Architecture → Trade-offs | `workflow-system-design` |
| System Docs | Reverse-engineer existing systems | Reconnaissance → Architecture discovery → APIs → Data → Infra → Audit | `workflow-system-docs` |
| Design Proposal | Redesigns, migrations, improvements | Current state → Gap analysis → Proposed design → Migration roadmap | `workflow-design-proposal` |
| Quick Fix | Bug fixes, patches, small changes | Understand → Branch & fix → Test → Ship | `workflow-quickfix` |
| Research | Investigation, analysis, evaluation | Define scope → Investigate → Synthesize → Document | `workflow-research` |
| Documentation | Guides, docs, runbooks | Scope → Analyze → Write → Review → Publish | `workflow-documentation` |
| Admin | Coordination, comms, processes | Plan → Execute → Follow up → Close | `workflow-admin` |

## Examples of user requests that trigger this skill

| User says | Action |
|---|---|
| "start task 683a..." | Start the task with the given ID |
| "start task 142" | Start the task with integer ID 142 |
| "work on the email template task" | Search for the task, then start it |
| "begin task" | Ask which task to start |
| "let's work on {task title}" | Search by title, confirm, then start |
| "I want to build a new API" | No task ID — create task in DailyPlanner first, then start |

## Critical Rules (Apply to ALL Workflows)

### 1. One Task Per Session
- ⛔ **NEVER work on multiple tasks in the same session**
- Each task gets its own Copilot session with name format: `{Integer Task ID}-{Task Title}`
- If a task is already active, require the user to start a new session

### 2. All Tasks Must Exist in Daily Planner
- ⛔ **NEVER start work without a Daily Planner task**
- If the user describes ad-hoc work, create the task in DailyPlanner first using `DailyPlanner-create_task`

### 3. Multi-Model Completion Review
Task completion reviews are **tiered by workflow type** to balance quality with speed:

| Tier | Workflows | Review Requirement |
|------|-----------|-------------------|
| **Full (mandatory)** | `engineering-task`, `workflow-system-design`, `workflow-design-proposal` | All 4 models must review before completion |
| **Standard (recommended)** | `workflow-quickfix`, `workflow-research`, `workflow-system-docs` | 4-model review recommended; can proceed with 2+ models if time-constrained |
| **Light (optional)** | `workflow-admin`, `workflow-documentation` | Single-model review sufficient; 4-model review available on request |

**The 4-model review panel:**

| Model | Agent Type | Focus |
|-------|-----------|-------|
| Claude Sonnet 4.6 | `general-purpose`, `model: "claude-sonnet-4.6"` | Completeness, problem-solution fit, edge cases |
| Claude Opus 4.6 | `general-purpose`, `model: "claude-opus-4.6"` | Overall quality, architecture, best practices |
| Gemini 3 Pro | `general-purpose`, `model: "gemini-3-pro-preview"` | Architecture coherence, scalability, patterns |
| GPT 5.4 | `general-purpose`, `model: "gpt-5.4"` | Security, performance, practical usability |

Run all 4 reviews in **parallel** before marking the task complete. Address any critical findings before completion.

**Review prompt template:**
```
Review the completed work for task: {task title}

Workflow: {workflow type}
Definition of Done:
{DoD criteria}

Work produced:
{summary of deliverables — code changes, documents, designs, etc.}

Check:
1. Does the work satisfy the Definition of Done?
2. Are there bugs, logic errors, or security concerns?
3. Is the work complete and thorough?
4. What improvements would you recommend?

Provide specific, actionable feedback only.
```

### 4. Code Quality Mandates
Any code produced during a task **must** meet these standards:

- ✅ **Documentation:** All functions, classes, and modules must have clear docstrings/comments explaining purpose, parameters, and return values. Non-obvious logic must have inline comments.
- ✅ **Unit tests:** All new code must have comprehensive unit tests covering happy path, edge cases (null, empty, boundary), and error paths. Aim for 80%+ coverage on new code.
- ✅ **Integration tests:** API endpoints and database interactions must have integration tests.
- ⛔ **NEVER ship code without tests** — if a test framework doesn't exist, set one up first.
- ⛔ **NEVER modify existing tests to make them pass without explicit user approval.**
- ✅ **UI testing:** Any change affecting the UI must be validated via `ui-testing-agent` MCP before shipping. UI test plan must be defined during design.

### 5. Git Branching & Pull Request Workflow
All code changes must go through branches and pull requests:

- ⛔ **NEVER push directly to main** — all work goes through feature/fix branches
- **Branch naming:** `feature/{integer-task-id}-{short-description}` or `fix/{integer-task-id}-{short-description}`
  - Check existing branches in the repo for naming conventions and follow them
- **Pull requests:** Create PR from feature/fix branch to `main` after code review
  - ⛔ **Do NOT auto-merge** — PRs require user review
  - Prompt the user to review: *"PR created at {URL}. Please review. Ask me to merge when ready, or merge manually."*
  - **Wait for user response** before proceeding
- **Post-merge:** Switch to `main` and pull latest:
  ```
  git checkout main && git pull origin main
  ```
  - ⛔ **Do NOT continue building on a merged branch** unless the user explicitly consents
  - Inform the user: *"Merged and switched to main. Do not continue on the old branch."*

### 6. Documentation Directory Convention
All task artifacts — plans, designs, technical docs, user docs, testing docs — must be stored in `system-documentation/` within the repository. This includes:
- Implementation plans
- Technical design documents
- Architecture decision records
- API documentation
- User guides
- Testing documentation
- Research findings
