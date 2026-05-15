---
name: start-task
description: >
  Start working on a Daily Planner task with a dedicated workspace, session, and
  workflow. Use this skill when the user says "start task", "work on task",
  "begin task", or references starting any task by ID. Sets up the workspace,
  detects the task type, and delegates to the appropriate workflow skill.
---

# Start Task — Orchestrator Skill

Sets up a dedicated workspace for each Daily Planner task, tracks it on the kanban board, and delegates to the appropriate workflow based on task type. Multiple tasks can be active in the same session — each is tracked independently on the board.

**⛔ Critical Rule: Every task is tracked on the board.** Every task gets its own board entry under `C:\boards\<board-name>\` and its own workspace. Multiple tasks may share a session; they must never share a board entry or workspace.

## Instructions

When the user wants to start working on a task, follow these steps in order:

### Step 0: Active Task Awareness

Before starting a new task, take stock of what's already active in this session **and across other sessions on the same board**:

1. **Check the board for active tasks**: read `inprogress.md` and find any entries whose `session.id` matches the current session.
2. **Check for cross-session activity on the same board**: scan **all** `inprogress.md` entries (regardless of session) for tasks whose `mutates.repos` overlaps the current task. These tasks may be running in another session — that's fine, **as long as each has its own worktree on its own branch** ([Parallel Tasks & Worktrees](#parallel-tasks--worktrees)).
3. **If one or more tasks are already active in this session:**
   - List them to the user with a brief status:
     ```
     ℹ️ This session is already tracking:
        • [142] Build Email Template — engineering-task, worktree C:\worktrees\<repo>\142-build-email-template
        • [87]  Fix Login Timeout    — quickfix,         worktree C:\worktrees\<repo>\87-fix-login-timeout
     
     Starting [{new id}] {new title} alongside the above. Each task gets its own
     board entry, worktree, branch, and activity log. Confirm to proceed, or open
     a fresh session if you'd prefer isolation.
     ```
   - Default to proceeding unless the user objects. If the user prefers isolation, guide them to a new session (`/exit`, `copilot`, `/rename {suggested session name}`).
4. **Conflict detection** — if any active task (this session OR another) has overlapping `mutates.paths` with the new task and the new task is marked `parallelizable: false`, warn:
   ```
   ⚠️ [142] in another session also mutates trading-backend/src/Email/**.
      The new task [273] is marked non-parallelizable on this path.
      Choose: queue [273] until [142] reaches inreview / override (run in parallel anyway) / cancel.
   ```
   See [Parallel Tasks & Worktrees](#parallel-tasks--worktrees) for the full conflict-resolution rules.
5. **Cross-task contamination is the real risk** — not co-existence. Make sure:
   - Each task has its own workspace directory (worktree for code repos).
   - Each task has its own board entry with its own session id stamped.
   - Activity logs and commits/PRs always reference the **specific** task they belong to (the board entry handles this).
6. **Proceed to Step 0.5.**

### Step 0.5: Resolve & Read the Task Board

Before touching DailyPlanner, locate the **task board** for the current work area, ensure its folder structure exists, and check whether the task already lives there.

1. **Resolve board root** (see [Board Conventions](#board-conventions) below):
   - `resolve-board-root()` returns `C:\boards\<board-name>\` for both code and non-code working directories.
   - The board name is derived from the `.board` pointer file → repo directory name → user-prompted fallback. See [Board Name Derivation](#board-name-derivation).
2. **Bootstrap the folder structure if missing** (idempotent — runs every invocation, no-ops if already complete):
   - Create the board folder if it doesn't exist.
   - Ensure all 5 stage files exist: `backlog.md`, `inprogress.md`, `inreview.md`, `blocked.md`, `completed.md`. Create any that are missing with their default header (e.g., `# In Progress\n\n> Tasks currently being worked on.\n`).
   - Ensure `README.md` exists; create from the template if missing.
   - **Run all 6 file existence checks + creations in parallel** — they are independent. See [Parallelization](#parallelization).
   - **If this was a brand-new board** (folder didn't exist before this run), continue to the [Seed & Migration](#seed--migration) flow to import existing DailyPlanner tasks.
3. **Look up the task** across all 5 stage files using the helper `read-task-from-board(taskId)` (greps the 5 files in parallel).
   - **If found in `inprogress.md`:** the task is already active — confirm with the user before resuming.
   - **If found in `backlog.md` / `inreview.md` / `blocked.md`:** use the embedded metadata; transition rules will move it later.
   - **If found in `completed.md`:** warn the user — they may be re-opening a closed task.
   - **If not found:** the task is new to this board; it will be added in Step 3.
4. **Run an on-demand sync** with DailyPlanner (`sync-with-daily-planner(taskId, mode=both)`) to refresh state and surface any conflicts before proceeding.

The board entry is the **operational source of truth** during a session; DailyPlanner remains the system of record. Always read the board first.

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

If this is the **first task in the session**, name it: `{Integer Task ID}-{Task Title}`

Examples:
- Task #142 "Build Email Template" → session name: `142-Build Email Template`
- Task #87 "Fix Login Timeout" → session name: `87-Fix Login Timeout`

If the session already hosts other active tasks (multi-task session):
- **Default:** keep the existing session name — don't rename mid-flight.
- **Optional:** suggest a batch label rename if the tasks share a clear theme:
  ```
  /rename multi-{theme} (e.g., multi-auth-fixes, multi-docs-update)
  ```
- The session name is for human navigation; **the board entry's `session.id` is what links activity to a task**, not the name.

If the user starts in a fresh session for a single task, include the rename command in the instructions.

### Step 1c: Review & Enrich (User Confirmation)

**Before any state changes happen** (no DailyPlanner status flip, no worktree, no stage transition), surface the full task to the user and give them an explicit opportunity to add context. This catches missing screenshots, late-breaking acceptance criteria, and lets the user back out if the task isn't really what they want to work on.

#### 1c.1 — Print the task

Render the task using the same shape as `review-backlog` Inspect mode ([review-backlog Inspect](../review-backlog/SKILL.md#mode-inspect)), reading from the board entry (which was located in Step 0.5). Include:

- Heading: `[<intId>] <title>`
- Trace ID, Priority, Type, Tags, Created date, Elaboration level
- `parallelizable` + `mutates.repos` + `mutates.paths`
- DailyPlanner status + ObjectId
- **Description** (full)
- **Images / References** — list every existing attachment with its caption + relative path; print `_(none)_` when empty
- **Definition of Done** — all four sub-sections (Acceptance Criteria, Validation Plan, Test Plan, Observability Plan), checked-state for each item, or `_Not yet elaborated._` placeholders
- **Notes & Decisions** (newest first, last 5)
- **Activity Log** (newest first, last 5)

End the block with a one-line summary of what's missing:
```
Missing for full elaboration: Validation Plan, Test Plan, Observability Plan, mutates.paths
```

#### 1c.2 — Ask one explicit confirmation question

```
ask_user:
  question: "About to start [{intId}] {title}. Anything to add or change first?"
  choices:
    - "Looks good — start (Recommended)"
    - "Add screenshots / reference images"
    - "Add notes or extra context"
    - "Adjust acceptance criteria"
    - "Fill in Validation / Test / Observability plans (recommended for P1/P2)"
    - "Hold off — back to the backlog"
```

The default ("Looks good") proceeds to Step 2 with no further questions. Each other choice triggers a sub-flow described below. After the chosen sub-flow finishes, return to this prompt until the user picks "Looks good" or "Hold off".

#### 1c.3 — Sub-flows

| Choice | Sub-flow |
|---|---|
| **Add screenshots / reference images** | Ask: *"Paste image paths separated by commas (or drop a folder path to take everything in it)."* For each valid image: copy into `C:\boards\<board>\attachments\<traceId>\<seq>-<slug>.<ext>`, append to YAML `attachments:`, append a markdown image ref to the **Images / References** body section, and call `record-activity(taskId, kind="attachment-added", payload={path, caption})`. See [Step 2b in add-to-backlog](../add-to-backlog/SKILL.md#step-2b-capture-reference-images-optional-but-encouraged-for-featuresbugs) for the exact path/slug rules. |
| **Add notes or extra context** | Free-form prompt: *"What's the extra context?"* Append a timestamped line to **Notes & Decisions** and `record-activity(taskId, kind="note", payload={text})`. Loop until user says "done". |
| **Adjust acceptance criteria** | Show current Acceptance Criteria list; ask *"Add / remove / reword?"*. Edit the **Acceptance Criteria** sub-section under Definition of Done. Update `elaboration.acceptanceCriteria` count. |
| **Fill in Validation / Test / Observability plans** | Hand off to `review-backlog`'s Plan mode ([Mode: Plan](../review-backlog/SKILL.md#mode-plan-flesh-out)) scoped to those three sub-sections. When it returns, update `elaboration.{validationPlanned,testPlanned,observabilityPlanned}` flags and the elaboration level. |
| **Hold off — back to the backlog** | Do nothing else. Log `{timestamp} — start-task aborted by user at Review & Enrich step` to the Activity Log. Exit the skill cleanly so the user can reconsider or pick a different task. |

Every sub-flow ends with `commit-board-change(<board>, "[<taskId>] start-task enrich: <action>")` so all pre-start enrichments are persisted to the boards repo before any state change happens.

#### 1c.4 — P1/P2 gate

If the user picks "Looks good" but the task is **P1 or P2** AND any of `validationPlanned` / `observabilityPlanned` is `false`, **warn before proceeding**:
```
⚠️ This is a P{1|2} item with no Validation Plan / Observability Plan yet.
   Starting without them means you'll have to bolt them on at PR time
   (where the pre-PR gate WILL block you).

   Choose: plan now (recommended) / start anyway / hold off
```
"Start anyway" is allowed but logged: `{timestamp} — Started without {missing sub-plans}; user overrode P1/P2 gate`.

Once the user confirms "Looks good" (and any P1/P2 gate has been resolved), proceed to Step 2.

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

### Step 3: Start the task — DailyPlanner + Board

1. Call `DailyPlanner-start_task` with the task ID to set status to "In Progress".
2. **Move the task entry on the board** using `move-task(taskId, fromStage, toStage="inprogress", contextUpdates)`:
   - If the task wasn't on the board, **create** the entry directly in `inprogress.md` populated from DailyPlanner.
   - If it was in `backlog.md`, cut it and append to `inprogress.md`.
   - Stamp these fields on the entry:
     - `session.id`, `session.name` (from current session)
     - `git.branch` (will be created in Step 6, may be left blank until then)
     - `timestamps.startedAt` (now, ISO-8601 UTC)
     - `stage: in-progress`
   - Append a line to `Activity Log`: `{timestamp} — Started; session={name}; workflow=<tbd>`
3. Push the updated entry back to DailyPlanner (`sync-with-daily-planner(taskId, mode=push)`) so status fields stay aligned.

### Step 4: Create the workspace directory

The workspace strategy depends on whether the task touches a code repository:

| Cwd type | Workspace |
|---|---|
| **Code repository** (parallelizable) | A **git worktree** at `C:\worktrees\<repo-name>\<task-id>-<short-title>\` on a fresh branch off `main`. Multiple tasks can run concurrently against the same repo with zero branch contention. See [Parallel Tasks & Worktrees](#parallel-tasks--worktrees). |
| **Code repository** (non-parallelizable) | Same as above, but a session-level lock is taken first to serialize against other in-progress tasks that share the same `mutates.paths`. |
| **Non-code cwd** | A plain workspace directory at `C:\repositories\personal-copilot-agent\<task-id>-<short-title>\` (no git, no worktree). |

1. **Build the directory name** using the session name format: `{Integer Task ID}-{Task Title}`
   - Replace spaces with hyphens, remove special characters, lowercase everything.
   - Example: Task #142 "Build Email Template" → `142-build-email-template`.
2. **For code repos** call `create-worktree(taskId)` ([Helper Operations](#helper-operations)):
   - Creates `C:\worktrees\<repo>\<dir-name>\` via `git worktree add` from the repo at `<repo-root>`.
   - Branch: `user/pingugi/<type-prefix>/<dir-name>` (`feature/` / `bugfix/` / `hotfix/` per [git-conventions](../../../instructions/git-conventions.instructions.md)). Base: `main`.
   - Stamps `session.worktree` and `session.branch` into the board entry.
   - Acquires a lock file at `C:\boards\<board>\.locks\<task-id>.lock` (gitignored).
3. **For non-code cwd** create the workspace directory at `C:\repositories\personal-copilot-agent\<dir-name>\` directly.
4. **Create a README.md** inside the workspace with the task context:

```markdown
# {Task Title}

**Task ID:** {taskId}
**Priority:** {priority}
**Type:** {type}
**Tags:** {tags}
**Due Date:** {dueDate or "None"}
**Workflow:** {detected workflow type}
**Worktree:** {session.worktree or "n/a"}
**Branch:** {session.branch or "n/a"}

## Description

{task description}

## AI Prompt

{AI prompt content if available, otherwise "No AI prompt configured for this task."}

## Documentation
All plans, designs, and documentation for this task live in `docs/` (organized by type: plans, features, decisions, etc.).

## Definition of Done

{Generated from task details — see Step 4b below. Mirror the four sub-sections (Acceptance Criteria / Validation Plan / Test Plan / Observability Plan) from the board entry.}
```

### Step 4b: Define the Definition of Done (DoD)

Every task must have a clear, measurable Definition of Done. Generate the initial DoD based on the task type, description, and workflow, then **ask the user to review and refine it**.

The **board entry is the canonical home for the DoD**. The workspace README.md transcludes (copies) the DoD from the board entry for convenience, but if they ever drift, the board entry wins. Update the DoD in the board entry first, then mirror it to README.md.

#### DoD Template
The Definition of Done lives in the **board entry** under `## Definition of Done` and is mirrored to the workspace README. It is **always structured into four sub-sections** — Acceptance Criteria, Validation Plan, Test Plan, Observability Plan — regardless of workflow.

```markdown
## Definition of Done

### Acceptance Criteria
- [ ] [Specific, user-visible, testable outcome derived from the task description]
- [ ] [Another criterion]
- [ ] [Another criterion]

### Validation Plan
- [ ] Manual / smoke validation steps (happy path)
- [ ] Manual / smoke validation steps (error path)
- [ ] Log lines we expect to see along the happy path (component → severity → message)
- [ ] Metric / dashboard checks that prove the feature is being exercised
- [ ] Failure-mode logs we expect to see when the error path triggers

### Test Plan
- [ ] Unit tests in `<test-project-or-file-path>` covering happy + error + edge cases
- [ ] Integration tests in `<test-project-or-file-path>` (if applicable)
- [ ] E2E / contract tests (only for consumer-facing surfaces)
- [ ] Regression test (for bugs) — failing pre-fix, passing post-fix

### Observability Plan
- [ ] Logs — list each required log line (level + structured fields + emitter) tied to a debug scenario
- [ ] Metrics — list each metric (name + type + labels) tied to the debug question it answers
- [ ] Traces — span names + key attributes for distributed flows
- [ ] Correlation — every log/metric/span carries the relevant correlation ID (TaskId, TraceId, PositionId, etc.)

### Quality Gates (always applied)
- [ ] Code reviewed (multi-model or peer review as appropriate)
- [ ] No known bugs introduced
- [ ] Documentation updated (if applicable)

### Workflow-Specific Criteria
{Generated based on detected workflow — see table below}
```

**P1/P2 items** captured via `add-to-backlog` must have **at minimum** Acceptance Criteria + Validation Plan + Observability Plan populated at capture. **P3/P4** may leave them as placeholders; `review-backlog`'s Plan mode fills them in for `full` elaboration. Workflows MUST NOT exit `inprogress → inreview` until all four sub-sections are populated and the listed logs/metrics are wired in code.

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
1. **At task start:** Generate or refresh the four DoD sub-sections from the board entry. Print all four sub-plans **up-front** before the workflow skill takes over, so the user sees the contract:
   ```
   📋 [142] Build Email Template — Definition of Done

   ✅ Acceptance Criteria (3)
      • Email template loader handles missing templates without crashing
      • Loader supports nested partials
      • Output passes Litmus rendering checks

   🔬 Validation Plan
      • Manual: send a render request for "welcome" — expect 200 + body
      • Log: INFO Email.TemplateLoader: loaded template=welcome bytes=…
      • Metric: email_template_render_total{status="success"} increments
      • Failure: render a missing template; expect WARN Email.TemplateLoader: template_missing

   🧪 Test Plan
      • Unit: tests/TradingManagement.Application.Tests/Email/TemplateLoaderTests.cs
      • Integration: tests/TradingManagement.Integration.Tests/Email/

   📊 Observability Plan
      • Logs: INFO loaded, WARN template_missing, ERROR template_invalid
      • Metrics: email_template_render_total (counter), _duration_ms (histogram)
      • Traces: Email.TemplateLoader.Load span with {template, source}
      • Correlation: carry TraceId + TaskId on every entry

   These are the pre-merge contract. Ask now if anything needs to change.
   ```
   Confirm with the user. If any sub-section is empty or marked placeholder, **stop and prompt** to populate before invoking the workflow skill.
2. **During task:** If scope changes or new requirements emerge, update the DoD in the board entry first, then mirror to README.md, and inform the user. Every edit goes through `commit-board-change()`.
3. **Pre-PR gate:** Before the workflow skill opens a PR (transition `inprogress → inreview`), verify each Observability-Plan log line / metric / span is actually wired in the diff:
   - Grep the diff for the literal log message strings declared in the Observability Plan.
   - Grep for the metric names + label sets declared.
   - If any are missing, **block the transition** with a clear message:
     ```
     ⛔ Observability Plan not satisfied for [142]:
        Missing log:   "Email.TemplateLoader: template_missing template={name}"
        Missing metric: email_template_render_total
        Wire these (or update the Observability Plan if intentionally dropped) before opening the PR.
     ```
4. **Before completion:** Review each DoD checkbox — all four sub-sections must have all items checked. Validation Plan items must include the literal log/metric checks the reviewer can run against staging.
5. **At completion:** Include DoD status in the completion summary, plus a one-line per sub-section ("Observability: ✅ 3 logs, 2 metrics, 1 span wired and confirmed in staging").

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

### 1. Every Task Tracked on the Board
- ⛔ **NEVER work on a task without a board entry** — the entry is created/moved at the start of every task
- Each task gets its **own workspace directory** and its **own board entry**, but **multiple tasks may share a session**
- Sessions can host parallel tasks; the board (`session.id` field on each entry) is what disambiguates which task an action belongs to
- Session naming defaults to the **first** task started: `{Integer Task ID}-{Task Title}`. When subsequent tasks join, optionally rename to a batch label (e.g., `multi-frontend-fixes`) at the user's discretion
- Cross-task contamination (sharing a workspace, conflating activity logs, mixing branches) is forbidden — the per-task board entry is the source of truth for what belongs to which task

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

### 6. Board is the Source of Active State
- ⛔ **NEVER** let a task's working state diverge from its board entry
- The board entry under `C:\boards\<board-name>\` is the **canonical operational view** of where a task is right now
- Every stage transition, commit, PR, decision, and notable action **must** be reflected in the board entry's metadata or `Activity Log`
- DailyPlanner remains the **system of record** for the task itself (priority, tags, description, due date)
- On every skill invocation that touches a task: read board → act → write board → sync DP → **`commit-board-change()`**. No exceptions.
- If the board and DailyPlanner disagree, follow the conflict rules in [Bi-directional Sync](#bi-directional-sync-with-dailyplanner) — never silently overwrite either side.

### 6b. Boards Repo Is Always Committed
- ⛔ **NEVER** leave a board mutation uncommitted at the end of a tool turn.
- Every mutation (entry created, stage moved, activity logged, attachment added, DoD edited, sync-back from DailyPlanner) flows through `commit-board-change()` ([Helper Operations](#helper-operations)).
- The boards repo at `C:\boards\` is a high-frequency, append-only audit log — readability of its `git log` is part of the contract. Commit messages follow `chore(<board>): [<taskId>] <action> — <reason>`.
- If a commit fails (no git identity, locked index, etc.) the helper surfaces the exact remediation command. The on-disk board write is still authoritative — the next successful mutation picks up the pending change.
- Pushing to a remote is **never automatic** unless the session has explicitly opted in. The local audit log is the default guarantee.

### 7. Documentation Directory Convention
All task **documentation artifacts** (specs, plans, migration notes) live under `docs/` inside the repository:
- Implementation plans, research plans, design proposals, system designs → `docs/plans/`
- Feature documentation → `docs/features/`
- Architecture decision records → `docs/decisions/`
- Bug fix documentation → `docs/bugs/`
- User guides → `docs/user-guides/`
- Task documentation (versioned upgrade/migration write-ups) → `docs/tasks/`
- Refactoring documentation → `docs/refactoring/`
- Change requests → `docs/change-requests/`

> **Note:** `docs/tasks/` here means *task documentation* (e.g., `v1.2-upgrade-to-net8.md`), **not** the kanban board. The kanban board lives at `C:\boards\<board-name>\` — see [Board Conventions](#board-conventions).

Use root-level `/docs/` for cross-project artifacts or project-level `{project}/docs/` for project-scoped work.

---

## Board System

The board is a markdown kanban that lives alongside the work it tracks. Every task this skill touches has exactly one entry on exactly one stage file at any time.

### Board Conventions

#### Location

Boards live **outside** the work repositories they track, in a single central directory:

```
C:\boards\
├── .git\                         — central boards repo (see "Central Boards Repository" below)
├── README.md                     — describes the boards-root convention
├── trade-management-system\      — one folder per board (matches repo name)
├── personal-copilot\             — non-code boards live here too
└── <other-board-name>\
```

Each board folder contains its own stage files and `attachments/` subfolder (see "Files" below).

Boards are intentionally **not** committed to the work repositories they track — keeps
private/incomplete planning out of public PRs and keeps board history independent of any
single repo's lifecycle.

#### Board Name Derivation

`resolve-board-name()` runs every invocation:

1. **`.board` pointer file at repo root or cwd** → if present, read the first line; use it
   verbatim if it points inside `C:\boards\`, else extract the board-name segment.
2. **Code repository (cwd has a `git rev-parse --show-toplevel`)** → use the **repo directory
   name** (`Split-Path -Leaf`) of the toplevel. Example: `C:\repositories\Sokokapu-Limited\trade-management-system`
   → board name `trade-management-system`. Owner is **not** included — boards are local-only.
3. **Non-code cwd** → ask the user for a board name once; default suggestion is the cwd
   basename. Cache the answer in the `.board` pointer file so we never ask again.

The full board path is `C:\boards\<board-name>\`.

#### Pointer File (`.board`)

Each tracked repository or working directory gets a one-line pointer file at its root:

```
C:\boards\<board-name>
```

- **File name**: `.board` (dotfile to keep it inconspicuous).
- **Location**: repo root (next to `.git`) or, for non-code cwd, the cwd itself.
- **Content**: exactly one line — the absolute path of the board folder.
- **Bootstrap**: created automatically by `resolve-board-root()` on first run if missing.
- **Commit policy**: safe to commit (it's a non-secret path marker). The skill never adds it
  to `.gitignore`. If the user prefers to keep it untracked, they may add `.board` to their
  global gitignore — the skill still treats the file as authoritative.
- **Validation**: on every invocation, if `.board` exists but points to a path that doesn't
  exist, warn the user and offer to recreate. If the pointed path is outside `C:\boards\`,
  honor it (lets advanced users place boards elsewhere) but log the deviation.

#### Central Boards Repository

`C:\boards\` itself is initialized as a single git repository the first time any board is
created. This gives one history, one backup target, and one place to sync from machine to
machine — without polluting any work repo.

- **Bootstrap**: on first board creation, if `C:\boards\.git` does not exist, run `git init`
  in `C:\boards\`, create a top-level `README.md` describing the convention, and commit it
  as `chore: init boards root`.
- **Layout**: each board is a sibling folder at `C:\boards\<board-name>\`. No nesting.
- **`.gitignore`** (at `C:\boards\.gitignore`):
  ```
  # Per-board ephemeral state
  */.locks/
  */attachments/.tmp/
  ```
  Locks are runtime-only (see Parallel Tasks). Attachments themselves **are** committed —
  reference images and screenshots are part of the task context.
- **Commit cadence — every mutation, no exceptions**: the boards repo is committed
  after **every** board write. There is no "small enough to skip" change.
  Mutations that trigger a commit:
  - Bootstrap of `C:\boards\` itself, a per-board folder, or a stage file.
  - Adding a new entry (`add-to-backlog`).
  - Stage transitions (`move-task`).
  - Activity log appends, notes, decisions, commit/PR records (`record-activity`).
  - Image/reference attachments added to a task.
  - DoD edits, YAML mutations, elaboration changes from `review-backlog`.
  - Sync result writes (when a `mode=pull` updated the board from DailyPlanner).

  Every mutation goes through the `commit-board-change()` helper (see [Helper Operations](#helper-operations)) which `git add -A` + `git commit -m <msg>` inside `C:\boards\`.
  Commit messages follow:
  ```
  chore(<board>): [<taskId>] <stage-or-action> — <short reason>
  ```
  Examples:
  - `chore(trade-management-system): [142] backlog → in-progress`
  - `chore(personal-copilot): [273] activity-log: pr-opened #47`
  - `chore(trade-management-system): [142] attach screenshot 1-dashboard.png`
  - `chore(boards-root): init boards root`

  Multiple mutations applied in a single tool turn (e.g., a batch transition of 5 tasks) are coalesced into **one commit per logical operation**, not one per file edit. The commit message lists the affected task IDs.
- **Remote (optional)**: the user may `git remote add origin <url>` themselves; the skill detects a configured remote and offers to push after each commit (default no). The skill never pushes without explicit per-session opt-in.

#### Files
```
C:\boards\<board-name>\
├── README.md             — format spec & conventions (auto-generated)
├── backlog.md            — not started, queued
├── inprogress.md         — actively being worked on
├── inreview.md           — PR open or awaiting review
├── blocked.md            — cannot proceed; waiting on something
├── completed.md          — done, kept for historical context
├── attachments\          — reference images, screenshots, per-trace subfolders
└── .locks\               — runtime worktree locks (gitignored)
```

A `_conflicts/` subfolder is created lazily to hold duplicates discovered during validation.

#### Stage File Layout
Each stage file is a flat list of task entries separated by `---`:

```markdown
# In Progress

> Tasks currently being worked on. Last synced: {timestamp}

---

## [142] Build Email Template
{YAML block + body}

---

## [87] Fix Login Timeout
{YAML block + body}
```

### Entry Format

A task entry is a heading plus a fenced YAML block plus prose sections. The heading uses the integer task ID for human scanning; the YAML `dailyPlanner.taskId` is the canonical key.

```markdown
## [{INT_ID}] {Task Title}

​```yaml
taskId: 142                                 # integer, display alias
traceId: tr-a8f3d2                          # short UUID, grep-able across systems
title: "Build Email Template"
priority: P2                                # P1|P2|P3|P4
type: Feature                               # Task|Feature|Product|Bug|UseCase|Scenario
tags: [engineering, frontend]
workflow: engineering-task                  # detected workflow skill (or proposed if not yet started)
stage: in-progress                          # backlog|in-progress|in-review|blocked|completed

parallelizable: true                        # see "Parallel Tasks & Worktrees"
mutates:
  repos: [trade-management-system]          # which board(s)/repo(s) this task touches
  paths:                                    # glob-like paths the task is expected to modify
    - "trading-backend/src/Email/**"
    - "trading-frontend/src/email-templates/**"

elaboration:
  level: full                               # minimal|partial|full
  estimatedEffort: medium                   # small|medium|large|null
  proposedWorkflow: engineering-task        # heuristic guess; null until elaborated
  acceptanceCriteria: 3                     # count of items in DoD; 0 means not yet defined
  validationPlanned: true                   # Validation Plan section populated
  testPlanned: true                         # Test Plan section populated
  observabilityPlanned: true                # Observability Plan section populated
  capturedBy: add-to-backlog                # which skill created this entry

session:
  id: 592a276e-26cd-483e-8822-298cb0e3d972
  name: "142-Build Email Template"
  worktree: "C:\\worktrees\\trade-management-system\\142-build-email-template"
  branch: "user/pingugi/feature/142-build-email-template"

git:
  branch: user/pingugi/feature/142-build-email-template
  baseBranch: main
  commits:                                  # accumulates as work progresses
    - sha: a1b2c3d
      message: "feat(email): add template loader"
      timestamp: "2026-05-15T11:42:00Z"
  pullRequests:
    - number: 47
      url: https://github.com/owner/repo/pull/47
      status: open                          # open|merged|closed
      openedAt: "2026-05-15T14:00:00Z"

dailyPlanner:
  taskId: "65a1f3..."                       # MongoDB ObjectId, canonical key
  outcomeId: null
  status: "In Progress"
  lastSyncedAt: "2026-05-15T10:30:00Z"      # null if never synced

timestamps:
  createdAt: "2026-05-14T09:00:00Z"
  startedAt: "2026-05-15T10:30:00Z"
  blockedAt: null
  reviewAt: null
  completedAt: null

filesTouched: []                            # repo-relative paths
relatedTasks: []                            # other taskIds this depends on / blocks
attachments:                                # reference images stored under attachments/<traceId>/
  - file: "attachments/tr-a8f3d2/1-mockup.png"
    caption: "Desired template layout from designer"
    addedAt: "2026-05-14T09:05:00Z"
​```

**Description**

{Verbatim description from DailyPlanner. Refreshed on every pull-sync.}

**Images / References**

Reference images stored under `C:\boards\<board>\attachments\<traceId>\`. Embed each via relative path so the entry renders in any markdown viewer rooted at the board folder.

![1-mockup.png — Desired template layout from designer](attachments/tr-a8f3d2/1-mockup.png)

**Definition of Done**

Definition of Done is **structured into four sub-sections** below. All four are required for `elaboration.level == "full"`. P1/P2 items must populate at least Acceptance Criteria + Validation Plan + Observability Plan at capture time; P3/P4 may leave them as placeholders for `review-backlog` to fill in.

### Acceptance Criteria
Specific, user-visible, testable outcomes. What proves the feature works for the end user / consumer?
- [ ] Acceptance criterion 1
- [ ] Acceptance criterion 2
- [ ] Acceptance criterion 3

### Validation Plan
How will we *verify in production / staging* that this change is actually doing what we intend? Validation is observable behaviour, not just "the test passed". For each acceptance criterion, list one or more validation steps:
- [ ] **Manual / smoke validation**: precise steps to reproduce success and failure paths.
- [ ] **Log checks**: the exact log line(s) — and severity — we expect to see along the happy path (e.g., `INFO Email.TemplateLoader: loaded template={name}`). Include the file/component that emits them.
- [ ] **Metric checks**: which dashboards / queries confirm the feature is being exercised in the wild (e.g., `email_template_render_total{status="success"}` increases after deploy).
- [ ] **Failure-mode validation**: deliberately trigger the error path and confirm the documented log/metric is emitted (e.g., `WARN Email.TemplateLoader: template_missing template={name}`).

### Test Plan
Pre-merge automation. List the test projects/files that must contain the new tests, with one bullet per test type used:
- [ ] **Unit tests**: `tests/TradingManagement.Application.Tests/Email/TemplateLoaderTests.cs` — cover happy path + missing-template + malformed-template.
- [ ] **Integration tests**: `tests/TradingManagement.Integration.Tests/Email/` — exercise the loader + renderer end-to-end against a real template.
- [ ] **E2E / contract / fixture tests**: only if the task is exposed to consumers (API, MT5, frontend).
- [ ] **Regression test** (for bugs): one failing test that reproduces the bug pre-fix; passes post-fix.

### Observability Plan
What instrumentation must be wired so this feature is **debuggable post-ship**. Tie each observable to a debug scenario.
- [ ] **Logs** — every required log line, with level + structured fields + the file that emits them. Example:
      - `INFO Email.TemplateLoader: loaded template={name} bytes={size}` (debug: "did the loader find the template?")
      - `WARN Email.TemplateLoader: template_missing template={name}` (debug: "did the consumer request an unknown template?")
- [ ] **Metrics** — counter/gauge/histogram name, type, labels, and the debug question each answers. Example:
      - `email_template_render_total{template, status}` counter — "are renders succeeding in prod?"
      - `email_template_render_duration_ms{template}` histogram — "is the renderer getting slow?"
- [ ] **Traces** — span name + key attributes for distributed flows. Example:
      - span `Email.TemplateLoader.Load` attrs `{template, source}` — "which call site is slow?"
- [ ] **Correlation** — every log/metric/span carries the relevant correlation ID (TaskId, TraceId, PositionId, etc.) so debugging across services is a single grep.

**Notes & Decisions**

- 2026-05-15T11:00 — Decided to use Handlebars over Liquid (simpler API, smaller bundle)

**Activity Log**

- 2026-05-15T10:30 — Started; session=142-Build Email Template; workflow=engineering-task; worktree=C:\worktrees\trade-management-system\142-build-email-template
- 2026-05-15T11:42 — Commit a1b2c3d on user/pingugi/feature/142-build-email-template
- 2026-05-15T14:00 — PR #47 opened
```

#### Schema Rules
- **Additive only** — never remove fields, only deprecate. Old entries must remain parseable.
- All timestamps are ISO-8601 UTC.
- Empty arrays/nulls are written explicitly (don't omit keys).
- Prose sections may be reordered, but the YAML block always comes immediately after the heading.

### Stage Transitions

| From → To | Trigger | Helper Call |
|---|---|---|
| (none) → backlog | Task created in DailyPlanner; sync pulls it in | `move-task(id, null, "backlog")` |
| backlog → inprogress | `start task {id}` invoked (Step 3) | `move-task(id, "backlog", "inprogress")` |
| inprogress → inreview | PR opened; user says "ready for review" | `move-task(id, "inprogress", "inreview")` |
| inreview → inprogress | Review requests changes | `move-task(id, "inreview", "inprogress")` |
| any → blocked | User says "blocked"; DP status set to blocked | `move-task(id, X, "blocked")` |
| blocked → previous | User says "unblocked" | `move-task(id, "blocked", priorStage)` |
| inreview → completed | PR merged + DP status Completed | `move-task(id, "inreview", "completed")` |

Every transition appends an entry to `Activity Log` and updates the relevant `timestamps.*` field.

### Bi-directional Sync with DailyPlanner

#### Field Ownership

| Field | Owner | Sync Direction |
|---|---|---|
| `priority`, `tags`, `description`, `dueDate` | DailyPlanner | Pull only (DP → board) |
| `git.*`, `session.*`, `filesTouched`, `Notes & Decisions`, `Activity Log` | Board | Push only (board → DP as comments/log) |
| `stage` ↔ DP `status`, `Definition of Done` checks | Both | Bi-directional |
| `dailyPlanner.outcomeId` | DailyPlanner | Pull only |

#### Conflict Resolution

Each entry tracks `dailyPlanner.lastSyncedAt`. On every sync:

1. Fetch DP state for the task; compare DP `updatedAt` vs board `lastSyncedAt`.
2. **DP unchanged since last sync** → push board → DP. Update `lastSyncedAt`.
3. **DP changed; board unchanged on shared fields** → pull DP → board. Update `lastSyncedAt`.
4. **Both changed on shared fields** → conflict:
   - Log a `Conflicts` entry under `Notes & Decisions` with both values and timestamps.
   - Default resolution: whichever field changed **most recently** wins.
   - Prompt the user to confirm if the difference is on `stage`/`status`.
   - Always push the resolved value to both sides; update `lastSyncedAt`.

#### Sync Modes

- `pull` — DP → board only. Used when reading state for display.
- `push` — board → DP only. Used after a board-driven change (transition, activity log).
- `both` — Full reconcile. Used at task start and on user demand.

### Helper Operations

These are the named operations the skill performs. They're documented here as logical functions; the skill executes them inline using the `view`/`edit`/`grep` tools and DailyPlanner MCP calls.

#### `resolve-board-root() → path`
1. **Resolve the board name** via `resolve-board-name()` (see [Board Name Derivation](#board-name-derivation)):
   - Prefer `.board` pointer file if present.
   - Else use the repo directory name (from `git rev-parse --show-toplevel`).
   - Else prompt the user (non-code cwd) and cache the answer in `.board`.
2. **Compute the board path**: `C:\boards\<board-name>\`.
3. **Bootstrap `C:\boards\` itself** if missing (see [Central Boards Repository](#central-boards-repository)): create the directory, run `git init`, write `README.md` + `.gitignore`. Call `commit-board-change(boards-root, "init boards root")`.
4. **Bootstrap the per-board folder**: create folder + 5 stage files + `attachments/` + `.locks/` + board-specific `README.md` in parallel if any are missing. Idempotent.
5. **Bootstrap the `.board` pointer file** at the repo root (or cwd) if missing — write the resolved path as the single line.
6. If the per-board folder didn't exist before this call → also run [Seed & Migration](#seed--migration).
7. Call `commit-board-change(<board>, "bootstrap board folder")` if any new file was created in step 4 or 5. (No-op if nothing changed.)
8. Return `C:\boards\<board-name>\`.

#### `read-task-from-board(taskId) → entry | null`
1. **In parallel**, grep all 5 stage files for the task: search each of `backlog.md`, `inprogress.md`, `inreview.md`, `blocked.md`, `completed.md` for `dailyPlanner.taskId: "<taskId>"` or `## [<intId>]` heading. Issue all 5 grep calls in a single tool batch.
2. If found, parse the YAML block + prose sections; return the entry plus its stage.
3. If found in **multiple** files → invoke conflict recovery (see [Edge Cases](#board-edge-cases)).
4. If not found, return `null`.

#### `move-task(taskId, fromStage, toStage, contextUpdates) → void`
1. Read the entry from `fromStage.md` (or generate a new one if `fromStage == null`).
2. Apply `contextUpdates` (merge into YAML; append to Activity Log).
3. Set `stage: <toStage>` and the relevant `timestamps.*` field.
4. Append a transition log line: `{timestamp} — Transition {fromStage} → {toStage}; {reason}`.
5. **Atomically**: append to `toStage.md` first, then remove from `fromStage.md`. (If a crash happens between, the duplicate-detection step on next read will reconcile.)
6. Call `sync-with-daily-planner(taskId, mode=push)`.
7. Call `commit-board-change(board, "[{taskId}] {fromStage} → {toStage}")`.

#### `sync-with-daily-planner(taskId, mode) → SyncResult`
1. Resolve the entry via `read-task-from-board`.
2. If `mode != push`: call `DailyPlanner-get_task(taskId)` and `DailyPlanner-get_task_prompt(taskId)`.
3. Apply field-ownership rules (see table above).
4. On conflicts: apply resolution rules; surface to user when needed.
5. If `mode != pull`: write changes back to DP via the appropriate `DailyPlanner-update_*` calls; for activity-log style updates use `DailyPlanner-add_activity_log`.
6. Update `dailyPlanner.lastSyncedAt` to now.
7. If the board was mutated by this call (mode `pull` or `both` produced writes), call `commit-board-change(board, "[{taskId}] sync from DailyPlanner")`.
8. Return `{ pulled, pushed, conflicts }`.

#### `record-activity(taskId, kind, payload) → void`
Appends to the entry without changing stage. Examples:

| `kind` | Payload | Effect |
|---|---|---|
| `commit` | `{ sha, message }` | Append to `git.commits`; activity log line |
| `pr-opened` | `{ number, url }` | Append to `git.pullRequests` (status=open); activity log line |
| `pr-merged` | `{ number }` | Set PR status=merged; activity log line; trigger transition to `completed` if DP status agrees |
| `file-touched` | `{ path }` | Add to `filesTouched` (deduped) |
| `note` | `{ text }` | Append timestamped line to `Notes & Decisions` |
| `decision` | `{ text }` | Append timestamped line to `Notes & Decisions` (prefixed `Decision:`) |
| `attachment-added` | `{ path, caption }` | Append to `Images / References`; activity log line |

After every call:
1. Push to DP via `sync-with-daily-planner(taskId, mode=push)`.
2. `commit-board-change(board, "[{taskId}] activity-log: {kind} {short-summary}")`.

#### `commit-board-change(board, message) → void`
The single chokepoint that persists every board mutation to the central boards repo. **Every helper above ends with a call to this function.**

1. `cd C:\boards\`.
2. `git add -A` — stage all changes across all per-board folders touched in this turn.
3. If `git diff --cached --quiet` returns true → **no-op** (no changes; skip the commit). This makes the helper safe to call after pure reads.
4. Otherwise `git commit -m "chore(<board>): <message>"`.
   - When the change spans multiple boards in one turn, use `chore(boards): <message>` and list affected boards in the body.
5. If `git remote get-url origin` succeeds **and** the session has opted-in to auto-push, run `git push origin <current-branch>`. Otherwise leave the commit local.
6. If `git commit` fails (e.g., no identity configured), warn the user once with the exact command to fix (`git -C C:\boards\ config user.email …` / `user.name …`) and continue without committing. The board entry change is still safe on disk; the next successful mutation will pick it up.

**Performance note:** when a single tool turn produces multiple mutations (e.g., a batch move of 5 tasks or a fan-out attachment write), call `commit-board-change` **once at the end** with a message summarizing the batch, not once per file edit. This keeps the boards-repo history readable.

#### `create-worktree(taskId) → path`
The single chokepoint for setting up a code-repo workspace. Used by Step 4 for every parallelizable task on a code repo.

1. Read the entry from `inprogress.md` (or wherever the task currently lives).
2. Resolve `<repo-root>` from the `.board` pointer file's sibling repo, or from the cwd's `git rev-parse --show-toplevel`. Compute `<repo-name>` = repo dir basename.
3. Compute paths and names:
   - `<dir-name>` = `<taskId>-<lowercase-hyphenated-title>` (truncate title at 40 chars).
   - `<worktree-path>` = `C:\worktrees\<repo-name>\<dir-name>\`.
   - `<branch-name>` = `user/pingugi/<type-prefix>/<dir-name>` (`feature/` | `bugfix/` | `hotfix/` per type).
4. Ensure `C:\worktrees\<repo-name>\` exists; create it if not.
5. Take a lock by writing `C:\boards\<board>\.locks\<taskId>.lock` with this session's id. If the lock already exists with a **different** session id, surface to the user (see [Parallel Tasks & Worktrees](#parallel-tasks--worktrees) for conflict handling).
6. From `<repo-root>` run `git fetch origin main` then `git worktree add -b <branch-name> <worktree-path> origin/main`. Use `--force` only after confirming an orphan worktree at the same path.
7. Stamp the board entry: `session.worktree`, `session.branch`, `git.branch`, `git.baseBranch: main`.
8. Append to Activity Log: `{timestamp} — Worktree created at {worktree-path} on {branch-name}`.
9. Call `commit-board-change(<board>, "[{taskId}] worktree created on {branch-name}")`.
10. Return `<worktree-path>`.

**Teardown** is symmetrical and invoked on stage transition `inreview → completed` (PR merged) or on explicit `start task --discard` requests:
1. From `<repo-root>` run `git worktree remove <worktree-path>` (or `--force` if dirty + user confirms).
2. Delete the lock file `C:\boards\<board>\.locks\<taskId>.lock`.
3. Activity log line: `{timestamp} — Worktree removed`. Commit.

### Parallel Tasks & Worktrees

Multiple tasks can run concurrently on the same code repository **without branch contention** because each task runs in its own [git worktree](https://git-scm.com/docs/git-worktree) on its own branch. The boards repo coordinates which task owns which paths so non-parallelizable tasks can be queued or detected at start time.

#### Why worktrees, not separate clones
- Shared `.git` directory → no disk-space duplication for large repos.
- Branch switching is instant; multiple branches checked out simultaneously.
- Native git tooling — no custom orchestration.
- Lock files at `C:\boards\<board>\.locks\` provide one extra layer of cross-session awareness.

#### Conflict matrix

| Active task `mutates.paths` | New task `mutates.paths` | New task `parallelizable` | Action |
|---|---|---|---|
| Disjoint | Disjoint | `true` | Proceed in parallel; create worktree |
| Overlapping | Overlapping | `true` (both) | Warn + proceed; user owns merge resolution |
| Overlapping | Overlapping | `false` (either) | **Queue** the non-parallel task until the active one reaches `inreview`, OR ask the user to override |
| Unknown (`paths` empty) | Anything | Anything | Treat as **non-parallelizable** for that task; conservative default |

`mutates.paths` should be populated at backlog capture (`add-to-backlog` infers it from the task description and tags) and refined during `review-backlog` Plan mode. When empty, the system assumes the task touches everything and serializes.

#### "Start parallel tasks" mode

Triggered by user phrases like *"start parallel tasks"*, *"pick the next N tasks"*, *"run parallel work"*:

1. Read all backlog entries that are `parallelizable: true` AND have a populated DoD (all four sub-sections).
2. Sort by priority desc, then age desc.
3. Pick the first **N** items whose `mutates.paths` sets are pairwise disjoint. Default `N = 3`; cap at the per-session limit configured in `session_state` (key: `parallelTaskLimit`).
4. For each selected task:
   - Run Step 3 (start on board + DP) and Step 4 (create worktree) **in parallel**.
   - Stamp `session.id` to the **current** session for all of them — they share one session for orchestration but each gets its own board entry, worktree, branch, and activity log.
5. Print a summary:
   ```
   🚀 Started 3 parallel tasks (trade-management-system):
      [142] Build Email Template       — C:\worktrees\trade-management-system\142-build-email-template
      [273] Add Pagination To Search   — C:\worktrees\trade-management-system\273-add-pagination-to-search
      [301] Document Risk Override     — C:\worktrees\trade-management-system\301-document-risk-override
   
   Switch between them with `cd <worktree-path>`. All board updates flow to C:\boards\trade-management-system\.
   ```
6. The user can `cd` into a worktree and work on one task; or open separate terminals/sessions per worktree for true parallel work. Each worktree's `git status` is independent.

#### Cross-session safety
- Every worktree creation acquires a `.locks/<taskId>.lock` file containing the owning session id.
- If a second session tries to `start task <same-id>`, Step 2 (resume support) detects the existing workspace **and** the lock; it offers to "join" the existing worktree (multi-window editing) or "take over" (reassign the lock to the new session, only if the prior session is no longer active).
- Locks are runtime-only and gitignored — they don't pollute the boards-repo history.

### Rolling Review

When many small tasks are running in parallel, reviewing them in big batches at the end loses context and creates a review queue bottleneck. The Rolling Review cadence keeps review in lockstep with implementation.

#### How it works

1. **Cadence is configurable per session**, stored in `session_state` (SQL):
   ```sql
   INSERT OR REPLACE INTO session_state (key, value) VALUES ('rollingReviewCadence', '3');
   ```
   Default = **3** tasks. Set to `0` to disable.
2. **Trigger**: whenever a task transitions to `inreview`, count the number of `inreview` entries on the board for this session. When the count hits the cadence threshold (default 3), pause new work and prompt:
   ```
   🔍 Rolling Review — 3 tasks ready for review:
      [142] Build Email Template     — PR #47 open
      [273] Add Pagination To Search — PR #48 open
      [301] Document Risk Override   — DoD checks pending
   
   Options:
     1. Review now (recommended) — walk the user through each task's DoD + PR
     2. Keep going — start more tasks; review later
     3. Adjust cadence (e.g., review every 2 or 5)
   ```
3. **Review walk-through** (option 1): for each task in `inreview`, surface:
   - Acceptance Criteria checklist (which boxes are checked).
   - Validation Plan results (logs/metrics observed in staging).
   - Test Plan summary (test files added, pass count).
   - Observability Plan compliance (logs/metrics/spans wired).
   - PR link with the latest review comments.
   - One-question prompt: *approve & merge / request changes / hold*.
4. **After review**, transition approved tasks `inreview → completed` (which removes their worktree and commits the closure). Tasks held remain in `inreview`; tasks needing changes go back to `inprogress`.
5. **Mid-review additions**: if the user starts a new task while a Rolling Review is in progress, warn them but allow it. The cadence threshold is computed on transitions, not start.

#### Why this beats end-of-batch review
- Review feedback informs the next 3 tasks before they're built.
- Context is fresh — the user just saw the task design 30 minutes ago.
- Worktrees and locks are released earlier, freeing parallelism slots.
- The boards repo's `git log` reads as a steady stream of `backlog → inprogress → inreview → completed` transitions in small batches.

### Legacy Board Migration

For repositories that still have a `<repo>/docs/tasks/` kanban from the pre-`C:\boards\` convention, run this migrator **once** on first `start-task` / `add-to-backlog` / `review-backlog` invocation.

#### Detection

In Step 0.5 (`resolve-board-root`), after computing `C:\boards\<board-name>\` but **before** seeding, check whether `<repo-root>/docs/tasks/` exists with any stage files (`backlog.md`, `inprogress.md`, `inreview.md`, `blocked.md`, `completed.md`). If yes, the migrator runs.

#### Steps

1. **Prompt the user** with a single confirmation:
   ```
   📦 Legacy kanban detected at C:\repositories\<repo>\docs\tasks\
      → Migrating to C:\boards\<board-name>\
        • Move:   5 stage files + README + any attachments
        • Update: setup-docs-standards docs/tasks/ becomes task documentation only
        • Pointer: write C:\repositories\<repo>\.board → C:\boards\<board-name>
   
   Confirm? (yes / preview-changes / skip)
   ```
2. **`preview-changes`** prints the file moves but does not execute.
3. **`yes`** executes the migration **idempotently**:
   - For each stage file in the legacy folder: if the destination is empty, move it; if the destination has entries, **append** the legacy entries (preserving each entry's YAML), and deduplicate by `taskId`. Duplicates land in `C:\boards\<board>\_conflicts\<taskId>-legacy.md` for user review.
   - Copy `attachments/` recursively if present.
   - Delete the legacy folder **only after** verifying every entry from the legacy stage files exists in the new location (or in `_conflicts/`).
   - Write the `.board` pointer file at the repo root.
4. **Validate the migration**: re-run `read-task-from-board` across the new stage files and confirm task counts match (legacy count == new count + conflicts count). On mismatch, **abort and restore** the legacy folder — do not delete anything until the user resolves the discrepancy.
5. **Commit the migration** with one summary commit:
   ```
   chore(<board>): migrate from <repo>/docs/tasks/ — N tasks moved
   ```
6. **Surface in setup-docs-standards**: print a one-line reminder that `docs/tasks/` in the repo now means *task documentation only*; the kanban lives at `C:\boards\<board>\`. See [Documentation Directory Convention](#7-documentation-directory-convention).

#### Migrator edge cases

| Situation | Handling |
|---|---|
| Legacy folder has no stage files | No-op — treat as empty pre-existing folder; ignore. |
| Destination already has entries with same `taskId` | Move the legacy entry to `_conflicts/<taskId>-legacy.md`; do not overwrite the destination. |
| Legacy entry has malformed YAML | Move it to `_conflicts/<taskId>-malformed.md`; record in activity log. |
| User says `skip` | Record the decision in `C:\boards\<board>\.migration-skipped` so we don't prompt again; commit the marker file. |
| Repo has both `docs/tasks/` and a `.board` pointer | Trust the pointer; ignore the legacy folder (it has already been migrated; warn once). |

### Observability Templates

Copy-pasteable snippets for the most common stacks in this codebase. Use as a starting point when filling in the Observability Plan; tailor names and labels to the feature.

#### .NET (Microsoft.Extensions.Logging + OpenTelemetry)

```csharp
// Structured log — happy path
_logger.LogInformation(
    "Email.TemplateLoader: loaded template={Template} bytes={Size}",
    name, content.Length);

// Structured log — warning path
_logger.LogWarning(
    "Email.TemplateLoader: template_missing template={Template}",
    name);

// Metric (System.Diagnostics.Metrics)
private static readonly Meter _meter = new("TradingManagement.Email");
private static readonly Counter<long> _renderTotal =
    _meter.CreateCounter<long>("email_template_render_total");
private static readonly Histogram<double> _renderDuration =
    _meter.CreateHistogram<double>("email_template_render_duration_ms");

_renderTotal.Add(1, new("template", name), new("status", "success"));
_renderDuration.Record(elapsedMs, new("template", name));

// Trace span (System.Diagnostics)
using var activity = _activitySource.StartActivity("Email.TemplateLoader.Load");
activity?.SetTag("template", name);
activity?.SetTag("source", source);
```

#### Node.js / TypeScript (pino + OpenTelemetry)

```typescript
import { logger } from "./logging";
import { metrics, trace } from "@opentelemetry/api";

const meter = metrics.getMeter("email-templates");
const renderTotal = meter.createCounter("email_template_render_total");
const renderDuration = meter.createHistogram("email_template_render_duration_ms");
const tracer = trace.getTracer("email-templates");

logger.info({ template: name, bytes: content.length }, "Email.TemplateLoader: loaded");
logger.warn({ template: name }, "Email.TemplateLoader: template_missing");

renderTotal.add(1, { template: name, status: "success" });
renderDuration.record(elapsedMs, { template: name });

await tracer.startActiveSpan("Email.TemplateLoader.Load", async (span) => {
  span.setAttribute("template", name);
  span.setAttribute("source", source);
  // … work …
  span.end();
});
```

#### Correlation IDs

Every log/metric/span MUST carry the relevant correlation IDs used in this codebase:

| Domain | Correlation ID(s) |
|---|---|
| TradingAgent flows | `PositionId`, `Symbol`, `Timeframe` |
| Orchestrator / EA bridge | `TraceId`, `PositionTicket`, `Symbol` |
| API requests | `TraceId`, `RequestId`, `UserId` |
| Backtesting / Optimizer | `RunId`, `StrategyName`, `Symbol` |
| Frontend / TradeView | `TraceId`, `SessionId`, `UserId` |

Wire them via the framework's ambient context (`ILogger` scopes / pino bindings / OTEL baggage) rather than manually passing them through every log statement.

### Seed & Migration

Run this once when the board folder is freshly created (the bootstrap in Step 0.5 only sets up empty files; this populates them).

1. **Bootstrap is already done** by Step 0.5: folder + 5 empty stage files + `README.md` exist.
2. **Import open DailyPlanner tasks** scoped to the current project's filter tags:
   - `DailyPlanner-get_tasks(status="all")` → returns the full list.
   - **Fan out task detail fetches in parallel**: for each task, issue `DailyPlanner-get_task` and `DailyPlanner-get_task_prompt` calls **in batches of 10** (issue 10 calls in a single tool batch, wait, then next 10). Don't fetch one at a time.
   - Bucket by status as detail fetches complete:
     - DP `New` → `backlog.md`
     - DP `In Progress` → `inprogress.md`
     - DP `Testing` → `inreview.md`
     - DP `Completed` (last 30 days only) → `completed.md`
   - Build entries (YAML + prose) with `git.*` and `session.*` left empty.
3. **Write the 4 stage files in parallel** (one `edit` call per non-empty stage file, all batched in one tool turn).
4. Print a one-line summary: `📋 Seeded board with {N} tasks ({backlog}/{inprogress}/{inreview}/{completed})`.

After seeding, proceed with normal Step 0.5 lookup.

### Parallelization

The board operations have many independent steps. **Always batch them into a single tool turn when they don't depend on each other.** Sequential calls waste round-trips.

| Operation | Parallelizable units | How |
|---|---|---|
| **Bootstrap folder** | 5 stage files + README check/create | One tool turn with 6 `view` (existence check) calls; for any missing, one tool turn with N `create` calls |
| **`read-task-from-board`** | Grep across 5 stage files | One tool turn with 5 `grep` calls (one per file) |
| **Seed/import from DP** | Per-task `get_task` + `get_task_prompt` | Batches of 10 parallel calls; up to 20 in flight per turn |
| **`sync-with-daily-planner` for many tasks** | Each task is independent | Batch up to 10 sync flows per turn |
| **`record-activity` (multiple events)** | Each `kind` is independent | Batch all writes to a stage file as a single edit; batch DP pushes |
| **Stage transitions for unrelated tasks** | Independent moves | Each `move-task` involves 2 file edits; batch unrelated moves |
| **DoD generation + workflow detection** | Both read the same task | Run after task data is loaded; can be done in one inference step |

#### Rules

- **Never** issue dependent calls in parallel (e.g., reading the entry, then writing it — these must be sequential).
- **Always** issue independent grep/view/create calls in the same tool turn.
- **Batch size cap**: 10 per turn for DP MCP calls (avoid rate limits); 5–6 for file system calls.
- If a parallel batch fails partially, retry only the failed units, not the whole batch.
- Cap parallel writes to the **same file** at 1 — file edits are inherently sequential per file, but writes to **different** files parallelize cleanly.

### Board Edge Cases

| Situation | Recovery |
|---|---|
| Task entry found in **2+ stage files** | Keep the entry with the most recent `timestamps.*` value. Move others to `_conflicts/<taskId>-<stage>.md` for review. Add a `Conflicts` note. |
| **DailyPlanner unreachable** | Continue with board operations. Set `dailyPlanner.lastSyncedAt: null` to mark as drift. Queue sync for next run; warn user. |
| **Git not initialized** in cwd | Treat as non-code cwd: ask for a board name and cache it in a `.board` pointer file at the cwd root. Continue with the standard `C:\boards\<board-name>\` flow. Leave all `git.*` fields empty. Skip git-dependent activity kinds. |
| **Task ID collision** across boards | Use `dailyPlanner.taskId` (ObjectId) as the canonical key — globally unique. Integer IDs are display-only. |
| **Manual edit broke YAML** | Surface a parse error with the file/line; ask the user to fix or restore. Do not silently rewrite. |
| **DP task deleted** but board entry exists | Move to `completed.md` with `Notes: "DailyPlanner task deleted on {date}"`. Do not auto-remove from the board. |
| **Stage file > 500 entries** | Recommend archiving the oldest 50% from `completed.md` to `completed-archive-YYYY-Q.md`. |

### Examples

#### Example 1: Brand-new task in a fresh repo

User: *"start task 273"* (cwd = `C:\repositories\sample-repo`, no `C:\boards\sample-repo\` exists yet)

1. **Step 0**: No active task — proceed.
2. **Step 0.5**:
   - `resolve-board-name()` → `sample-repo` (from repo dir name).
   - `resolve-board-root()` → `C:\boards\sample-repo\` (doesn't exist).
   - Bootstrap `C:\boards\` itself (first board ever): `git init`, write README/.gitignore, commit `chore: init boards root`.
   - Bootstrap the per-board folder: create folder + 5 stage files + `attachments/` + `.locks/` + README.
   - Write `.board` pointer file at `C:\repositories\sample-repo\.board` with `C:\boards\sample-repo`.
   - **Seed**: import open DP tasks; task 273 lands in `backlog.md`.
   - `read-task-from-board(273)` → found in `backlog.md`.
   - `sync-with-daily-planner(273, mode=both)` → no drift.
3. **Step 1**: Already have full DP details from sync.
4. **Step 1b**: Session name = `273-Add Pagination To User Search`.
5. **Step 3**:
   - `DailyPlanner-start_task(273)`.
   - `move-task(273, "backlog", "inprogress", { session, timestamps.startedAt })`.
   - Activity log: `2026-05-15T10:50 — Started; session=273-Add Pagination To User Search; workflow=engineering-task`.
6. **Step 4**: Workspace at `C:\repositories\personal-copilot-agent\273-add-pagination-to-user-search\`.
7. **Step 4b**: DoD generated from task description; written to board entry; mirrored to workspace README.
8. **Step 5–8**: Detect `engineering-task`; invoke skill.

#### Example 2: Resuming a blocked task across sessions

User: *"start task 142"* (cwd = repo with existing board; entry is in `blocked.md`)

1. **Step 0**: No active task — proceed.
2. **Step 0.5**:
   - `read-task-from-board(142)` → found in `blocked.md`. Embedded note: `"Waiting on infra team for Redis access"`.
   - `sync-with-daily-planner(142, mode=both)` → DP status changed to "In Progress" yesterday (unblock).
   - **Conflict detected** on `stage` (board=blocked, DP=In Progress). User is prompted; chooses to unblock.
3. **Step 3**:
   - `move-task(142, "blocked", "inprogress", { timestamps.blockedAt: null, note: "Unblocked: Redis access granted" })`.
   - Activity log shows the original block reason and the unblock — full continuity.
4. Workspace already exists from prior session → README unchanged; resume from recorded `currentPhase`.
5. Workflow skill picks up where it left off.
