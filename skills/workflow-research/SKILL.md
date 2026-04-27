---
description: "Research and investigation workflow for exploring topics, analyzing systems, and synthesizing findings. Use this skill when the user says 'research', 'investigate', 'explore', 'analyze', 'deep dive', 'study this', or when a task is tagged 'research'. Produces structured findings with recommendations and follow-up actions."
---

> ⚠️ **Prerequisite:** This workflow must be started via the `start-task` skill to ensure session isolation, workspace setup, and task tracking. If invoked directly, say: "start task [task description]" instead.

# Research Workflow

## Context
Research tasks require a different approach than building features. The goal is to understand, analyze, and document — not to ship code. This workflow structures the investigation process to produce actionable findings that inform future decisions and tasks.

## When to Use
- Investigating a technology, pattern, or approach
- Analyzing existing systems or codebases
- Exploring solutions to a problem before committing to implementation
- Understanding team processes, architectures, or dependencies
- Any task where the primary deliverable is knowledge, not code

## Workflow

### Phase 1: Define Scope
1. **Clarify research questions:**
   - What specific questions need to be answered?
   - What decisions will this research inform?
   - What is the expected output format?

2. **Set boundaries:**
   - Time-box the research (agree with user on depth)
   - Define what's in-scope and out-of-scope
   - Identify key stakeholders who need the findings

3. **Create research outline:**
   Save to `system-documentation/research-plan.md`:
   ```markdown
   # Research: [Topic]
   
   ## Questions to Answer
   1. [Primary question]
   2. [Secondary questions]
   
   ## Scope
   - In: [what we'll investigate]
   - Out: [what we won't cover]
   
   ## Sources to Check
   - [ ] Codebase analysis
   - [ ] Internal documentation (eng.ms)
   - [ ] Team knowledge (Notion)
   - [ ] External resources
   ```

### Phase 2: Investigate
Use multiple sources in parallel:

1. **Codebase exploration:**
   - Use `explore` agents to analyze code patterns, dependencies, and architecture
   - Trace data flows and call graphs
   - Identify relevant files and components

2. **Internal documentation:**
   - Search `eng.ms` via `enghub-search` for TSGs, team docs, and guides
   - Search Notion for related notes, ADRs, and meeting notes
   - Check existing design documents

3. **External resources:**
   - Use `web_search` for current best practices, comparisons, and community insights
   - Review official documentation for technologies being evaluated

4. **Data analysis (if applicable):**
   - Query databases or APIs for usage data
   - Analyze logs, metrics, or telemetry

### Phase 3: Synthesize
1. **Organize findings:**
   Create `system-documentation/findings.md`:
   ```markdown
   # Research Findings: [Topic]
   
   ## Summary
   [2-3 paragraph executive summary]
   
   ## Key Findings
   ### Finding 1: [Title]
   [Evidence, data, and analysis]
   
   ### Finding 2: [Title]
   [Evidence, data, and analysis]
   
   ## Comparison Matrix (if evaluating options)
   | Criteria | Option A | Option B | Option C |
   |----------|----------|----------|----------|
   | [Criterion 1] | [Assessment] | [Assessment] | [Assessment] |
   
   ## Recommendations
   1. [Primary recommendation with rationale]
   2. [Secondary recommendations]
   
   ## Open Questions
   - [Questions that couldn't be answered]
   
   ## References
   - [Links to sources used]
   ```

2. **Validate findings:**
   - Cross-reference findings across sources
   - Identify contradictions or gaps
   - Note confidence levels for each finding

### Phase 4: Document and Share
1. **Save to Notion (if applicable):**
   - Create a Notion page with the findings
   - Link to the Daily Planner task
   - Tag relevant teams or topics

2. **Save locally:**
   - Ensure findings are committed to the workspace repo
   - Update README.md with summary and links

3. **Create follow-up tasks:**
   - If research reveals actionable items, create tasks in Daily Planner
   - Link follow-up tasks to the research findings
   - Tag follow-ups with appropriate workflow type (engineering, docs, etc.)

4. **Run mandatory completion review:**
   Run the 4-model completion review as defined in `start-task` Critical Rules §3 (Sonnet, Opus, Gemini, GPT-5.4 in parallel). Address critical findings before marking done.

5. **Complete the task:**
   ```
   DailyPlanner-complete_task(taskId: "[task_id]", summary: "[key findings summary]")
   ```

## Integration Points
- **Learning Notes:** If research teaches new concepts, suggest `learning-notes` skill to capture them
- **Architecture Decisions:** If research leads to a decision, suggest `architecture-decision` skill
- **Engineering Task:** If research leads to implementation work, suggest creating a new engineering task
- **Activity Log:** Log research progress via `DailyPlanner-add_activity_log`

## Graceful Fallback
- If DailyPlanner is unavailable, continue the research without task tracking — log progress locally in the workspace README
- If Notion is unavailable, save all findings and documents locally in the workspace instead
- If external tools fail (eng.ms, WorkIQ, web_search), proceed with available sources and note the gap
- If a phase cannot be completed, document the blocker in the workspace and skip to the next actionable phase

## Rules
1. ✅ Always start with clear research questions
2. ✅ Use multiple sources — don't rely on a single source
3. ✅ Document findings even if inconclusive
4. ✅ Create follow-up tasks for actionable findings
5. ⛔ Don't jump to implementation — this workflow produces knowledge, not code
6. ✅ Mandatory 4-model completion review before marking done (see `start-task` Critical Rules §3)
7. ✅ Any code produced must be documented and have unit tests
