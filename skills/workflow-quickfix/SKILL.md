---
description: "Quick fix workflow for small bug fixes and minor changes. Use this skill when the user says 'quick fix', 'bug fix', 'hotfix', 'small fix', 'patch', or when a task is tagged 'quickfix'. Streamlined branch → fix → test → PR workflow without design documentation."
---

> ⚠️ **Prerequisite:** This workflow must be started via the `start-task` skill to ensure session isolation, workspace setup, and task tracking. If invoked directly, say: "start task [task description]" instead.

# Quick Fix Workflow

## Context
Not every task needs a full engineering lifecycle. Quick fixes — bug patches, config tweaks, small UI corrections — benefit from a streamlined workflow that gets changes shipped fast while maintaining quality. This workflow skips design documentation and plan-phase multi-model review in favor of speed. A single code-review pass replaces the full design review. The completion review is recommended but not mandatory for quickfixes (see `start-task` Critical Rules §3).

## Scope Guard
This workflow is appropriate when:
- The change touches **1-3 files** max
- The fix is **under ~100 lines** of changed code
- The root cause is **understood** or quickly identifiable
- No **architectural decisions** are needed

If the fix grows beyond this scope, escalate to the `engineering-task` skill.

## Workflow

### Phase 1: Understand the Issue
1. **Reproduce the problem:**
   - Get clear reproduction steps from the user or task description
   - Identify the affected component/file

2. **Root cause analysis:**
   - Use the `explore` agent to trace the issue in the codebase
   - Identify the exact code path causing the problem
   - Check if there are related issues or previous fixes

3. **Confirm scope:**
   - If the fix requires changes to more than 3 files or involves architectural changes, recommend escalating to `engineering-task`

### Phase 2: Branch and Fix
1. **Create a fix branch:**
   ```
   git checkout -b fix/{integer-task-id}-{short-description}
   ```
   ⛔ **NEVER push directly to main.**

2. **Implement the fix:**
   - Make the minimal change needed to resolve the issue
   - Follow existing code conventions
   - Add inline comments only if the fix is non-obvious

3. **Add a regression test:**
   - Write a test that would have caught this bug
   - Ensure the test fails without the fix and passes with it
   - If no test framework exists for the affected code, note this as a gap

### Phase 3: Verify
1. **Run existing tests:**
   - Run the project's test suite to ensure no regressions
   - Run linters/formatters if configured

2. **Manual verification:**
   - Verify the original reproduction steps no longer produce the bug
   - Check edge cases around the fix

3. **Single-model code review:**
   - Use a `code-review` agent to review the diff
   - Address any issues found (bugs, security, logic errors)

4. **UI validation (if fix touches the UI):**
   - Start local backend and frontend
   - Use `ui-testing-agent` MCP to validate:
     - The original bug no longer reproduces in the UI
     - Related UI functionality still works (regression check)
     - If Google/OAuth login is needed, prompt the user to complete it
   - ⛔ Do not skip UI validation for UI-affecting fixes

### Phase 4: Ship
1. **Commit with descriptive message:**
   ```
   git add -A && git commit -m "fix: {short description}

   {root cause explanation}
   {what the fix does}

   Co-authored-by: Copilot <223556219+Copilot@users.noreply.github.com>"
   ```

2. **Run mandatory completion review:**
   Run the 4-model completion review as defined in `start-task` Critical Rules §3. Address critical findings.

3. **Push and create PR to main:**
   ```
   git push -u origin fix/{integer-task-id}-{short-description}
   ```
   Create a pull request from the fix branch to `main`.
   ⛔ **Do NOT auto-merge.**

4. **Prompt user to review:**
   ```
   ✅ Pull request created: {PR URL}
   📋 Please review the PR. Ask me to merge when ready, or merge manually.
   ```
   Wait for user response.

5. **Post-merge:** Switch to main:
   ```
   git checkout main && git pull origin main
   ```
   ⛔ Do NOT continue on the merged branch without user consent.

6. **Complete the task:**
   ```
   DailyPlanner-complete_task(taskId: "[task_id]", summary: "[what was fixed]")
   ```

## Integration Points
- **Engineering Checklist:** Before shipping, verify `engineering-checklist` §1 (Security — input validation) and §2 (Testing — regression test added)
- **Impact Tracker:** If task is tagged "official", suggest documenting impact on completion
- **Activity Log:** Log fix completion via `DailyPlanner-add_activity_log`

## Graceful Fallback
- If DailyPlanner is unavailable, continue the fix without task tracking — log progress locally in the workspace README
- If Notion is unavailable, save all notes and documents locally in the workspace instead
- If external tools fail (eng.ms, WorkIQ, web_search), proceed with available sources and note the gap
- If a phase cannot be completed, document the blocker in the workspace and skip to the next actionable phase

## Rules
1. ✅ Always add a regression test if a test framework is available
2. ✅ Run existing tests before committing
3. ✅ All fix code must be documented (docstrings, inline comments for non-obvious logic)
4. ✅ Mandatory 4-model completion review before marking done (see `start-task` Critical Rules §3)
5. ✅ If the fix involves documentation changes, update files in `system-documentation/`
6. ⛔ If scope exceeds 3 files or ~100 lines, escalate to engineering-task
7. ⛔ NEVER modify existing tests to make them pass without asking
8. ⛔ NEVER ship code without tests
