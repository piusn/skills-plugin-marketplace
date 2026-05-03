---
description: "Documentation workflow for creating and updating technical docs, user guides, and knowledge articles. Use this skill when the user says 'write docs', 'update documentation', 'create guide', 'document this', or when a task is tagged 'docs' or 'documentation'. Orchestrates analysis, writing, review, and publishing."
---

> ⚠️ **Prerequisite:** This workflow must be started via the `start-task` skill to ensure session isolation, workspace setup, and task tracking. If invoked directly, say: "start task [task description]" instead.

# Documentation Workflow

## Context
Good documentation is as important as good code. This workflow structures the documentation process — from understanding what needs documenting, through writing and review, to publishing in the right location. It integrates with existing documentation skills for heavy lifting.

## When to Use
- Writing new technical documentation or user guides
- Updating existing docs to reflect code changes
- Creating onboarding guides or runbooks
- Writing API documentation
- Any task where the primary deliverable is documentation

## Workflow

### Phase 1: Scope and Plan
1. **Identify documentation type:**
   - **API Reference** — endpoint specs, request/response formats
   - **Technical Guide** — architecture, design patterns, system overview
   - **User Guide** — how-to, getting started, feature walkthroughs
   - **Onboarding Guide** — new team member setup and context
   - **Runbook/TSG** — operational procedures, troubleshooting
   - **ADR** — architecture decision record (delegate to `architecture-decision` skill)

2. **Identify the audience:**
   - Developers on the team?
   - New team members?
   - External users/consumers?
   - Operations/on-call engineers?

3. **Gather requirements:**
   - What must be covered?
   - What existing docs exist? (check eng.ms, Notion, repo READMEs)
   - What format is needed? (Markdown, DocFX, Word, PowerPoint)

### Phase 2: Analyze Source Material
1. **Codebase analysis:**
   - Use `explore` agents to understand the code being documented
   - Identify public APIs, configuration options, and key flows
   - Read existing inline documentation and comments

2. **Existing documentation audit:**
   - Search `eng.ms` via `enghub-search` for related docs
   - Search Notion for team notes and meeting records
   - Check repo for existing READMEs, wikis, or doc folders

3. **Knowledge gaps:**
   - Identify areas where documentation is missing or outdated
   - Note questions to ask the team or user

### Phase 3: Write
1. **Draft the documentation:**
   - Follow the appropriate template for the doc type
   - Use clear, concise language appropriate for the audience
   - Include code examples, diagrams (Mermaid), and screenshots where helpful
   - Cross-reference related documentation

2. **For heavy documentation tasks**, delegate to the `tech-docs` skill which handles:
   - Multi-section technical documents
   - DocFX-formatted documentation
   - Comprehensive API references

3. **Structure guidelines:**
   - Lead with the most important information
   - Use progressive disclosure (overview → details → advanced)
   - Include a "Quick Start" or "TL;DR" section for long docs
   - Add a table of contents for documents > 3 sections

### Phase 4: Review
1. **Self-review checklist:**
   - [ ] Accurate — does it match the current code/system?
   - [ ] Complete — are all necessary topics covered?
   - [ ] Clear — would a new team member understand this?
   - [ ] Actionable — can readers follow the steps?
   - [ ] Consistent — does it match existing doc style?

2. **Technical accuracy review:**
   - Cross-reference code examples against the actual codebase
   - Verify all commands and steps work as written
   - Check links are valid

### Phase 5: Publish
1. **Determine publish location:**
   - **Repository** — commit to the appropriate `docs/` subdirectory within the repository:
     - Plans → `docs/plans/`
     - Feature docs → `docs/features/`
     - User guides → `docs/user-guides/`
     - Architecture decisions → `docs/decisions/`
   - **Notion** — create/update Notion pages for team knowledge
   - **eng.ms** — submit to EngineeringHub if it's a TSG or team doc
   - **Export** — use `markdown-to-word` or `md2pptx` for Office formats

2. **Commit and link:**
   - Commit documentation to the repo under the appropriate `docs/` subdirectory
   - Link from relevant code files (README references)
   - Update any doc indexes or navigation

3. **Run mandatory completion review:**
   Run the 4-model completion review as defined in `start-task` Critical Rules §3 (Sonnet, Opus, Gemini, GPT-5.4 in parallel). Address critical findings before marking done.

4. **Complete the task:**
   ```
   DailyPlanner-complete_task(taskId: "[task_id]", summary: "[docs created/updated]")
   ```

## Integration Points
- **System Docs:** For documenting an entire existing system, delegate to `workflow-system-docs` — it's structured for full codebase analysis with architecture diagrams and engineering audit
- **Tech Docs Skill:** Delegate heavy documentation generation to `tech-docs`
- **Markdown to Word:** Use `markdown-to-word` skill for Word exports
- **MD to PPTX:** Use `md2pptx` skill for presentation exports
- **Onboarding Guide:** Use `onboarding-guide` skill for team onboarding docs
- **Activity Log:** Log documentation progress via `DailyPlanner-add_activity_log`

## Graceful Fallback
- If DailyPlanner is unavailable, continue the documentation workflow without task tracking — log progress locally in the workspace README
- If Notion is unavailable, save all documents locally in the workspace instead
- If external tools fail (eng.ms, WorkIQ, web_search), proceed with available sources and note the gap
- If a phase cannot be completed, document the blocker in the workspace and skip to the next actionable phase

## Rules
1. ✅ Always check for existing documentation before writing from scratch
2. ✅ Validate technical accuracy against the actual codebase
3. ✅ Use the appropriate format for the audience and platform
4. ✅ Include code examples for technical documentation
5. ⛔ Don't publish without reviewing for accuracy
6. ✅ Mandatory 4-model completion review before marking done (see `start-task` Critical Rules §3)
