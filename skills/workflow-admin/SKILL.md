---
description: "Administrative and coordination workflow for non-engineering tasks like communications, meeting prep, reviews, and team coordination. Use this skill when the user says 'coordinate', 'organize', 'prepare', 'follow up', 'send update', or when a task is tagged 'admin' or 'coordination'. Plans, executes, and follows up on administrative deliverables."
---

> ⚠️ **Prerequisite:** This workflow must be started via the `start-task` skill to ensure session isolation, workspace setup, and task tracking. If invoked directly, say: "start task [task description]" instead.

# Administrative / Coordination Workflow

## Context
Not all tasks produce code. Administrative work — drafting communications, coordinating across teams, preparing reviews, organizing events — requires its own structured approach. This workflow ensures administrative tasks are completed thoroughly with proper follow-up.

## When to Use
- Drafting emails, status updates, or communications
- Coordinating across teams or stakeholders
- Preparing for reviews or presentations
- Organizing events, sessions, or meetings
- Managing processes or procedures
- Any task where the deliverable is coordination, communication, or organization

## Workflow

### Phase 1: Plan
1. **Define deliverables:**
   - What is the concrete output? (email, document, meeting, process)
   - Who are the stakeholders?
   - What is the deadline?

2. **Gather context:**
   - Use `workiq-ask_work_iq` to check relevant emails and calendar
   - Search Notion for related notes, previous communications
   - Check Daily Planner for related tasks

3. **Create action checklist:**
   Break the task into concrete, checkable steps:
   ```markdown
   ## Action Items
   - [ ] Gather information from [source]
   - [ ] Draft [deliverable]
   - [ ] Review with [stakeholder]
   - [ ] Send/publish [deliverable]
   - [ ] Follow up by [date]
   ```

### Phase 2: Execute
1. **Draft deliverables:**
   - Write emails, documents, or presentations
   - Prepare meeting agendas or talking points
   - Create spreadsheets, trackers, or process documents

2. **Leverage existing skills:**
   - **Meeting prep** → use `meeting-prep` skill
   - **Manager updates** → use `prep-duncan` or `prep-tarik` skills
   - **Status reports** → use `weekly-status` skill
   - **Presentations** → use `md2pptx` skill
   - **Word documents** → use `markdown-to-word` skill

3. **Coordinate with stakeholders:**
   - Share drafts for review
   - Incorporate feedback
   - Align on timelines

### Phase 3: Follow Up
1. **Track responses:**
   - Note who has responded and who hasn't
   - Send reminders if needed
   - Escalate if blockers are identified

2. **Update status:**
   - Log progress via `DailyPlanner-add_activity_log`
   - Update task description with current state

3. **Handle dependencies:**
   - If the admin task spawns other tasks, create them in Daily Planner
   - Link related tasks together

### Phase 4: Close
1. **Verify completion:**
   - All deliverables sent/published?
   - All stakeholders acknowledged?
   - Any follow-up actions documented?

2. **Document outcomes:**
   - Save final versions of communications or documents
   - Note decisions made and their rationale
   - Record any commitments or timelines agreed upon

3. **Run mandatory completion review:**
   Run the 4-model completion review as defined in `start-task` Critical Rules §3 (Sonnet, Opus, Gemini, GPT-5.4 in parallel). Address critical findings before marking done.

4. **Complete the task:**
   ```
   DailyPlanner-complete_task(taskId: "[task_id]", summary: "[what was coordinated/delivered]")
   ```

## Integration Points
- **Meeting Prep/End:** Use `meeting-prep` and `meeting-end` skills for meeting-related admin
- **Manager Updates:** Use `prep-duncan` and `prep-tarik` for management communications
- **Weekly Status:** Use `weekly-status` for recurring reports
- **Team Feedback:** Use `team-feedback` for recognition and contribution tracking
- **Impact Tracker:** If task is tagged "official", document impact on completion
- **Activity Log:** Log progress via `DailyPlanner-add_activity_log`

## Graceful Fallback
- If DailyPlanner is unavailable, continue the workflow without task tracking — log progress locally in the workspace README
- If Notion is unavailable, save all documents and notes locally in the workspace instead
- If external tools fail (eng.ms, WorkIQ, web_search), proceed with available sources and note the gap
- If a phase cannot be completed, document the blocker in the workspace and skip to the next actionable phase

## Rules
1. ✅ Always define concrete deliverables before starting
2. ✅ Leverage existing skills instead of duplicating their work
3. ✅ Document outcomes and decisions for future reference
4. ✅ Follow up on pending items — don't let things drop
5. ⛔ Don't over-engineer admin tasks — keep it pragmatic
6. ✅ Mandatory 4-model completion review before marking done (see `start-task` Critical Rules §3)
