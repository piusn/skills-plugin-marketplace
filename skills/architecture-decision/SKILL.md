---
description: "Document architecture decisions as Architecture Decision Records (ADRs). Use this skill when the user says 'document ADR', 'architecture decision', 'design decision', 'record decision', 'ADR', or 'why did we choose'. Creates structured decision records in Notion linked to relevant tasks."
---

# Architecture Decision Record (ADR) Skill

## Context
As Teams Architect, Pius makes architecture decisions regularly. ADRs capture the context, decision, and consequences so future engineers understand why choices were made. These are stored in Notion for team visibility.

## When to Use
- When making a significant architecture or design decision
- When documenting a decision that was already made
- When reviewing past decisions
- During design phases of engineering tasks

## Workflow

### Step 1: Gather Decision Context
Ask the user for the decision details:
```
ask_user: "What architecture decision do you want to document?"
```

Then gather:
- **Title:** Short descriptive name
- **Context:** What situation prompted this decision?
- **Options considered:** What alternatives were evaluated?
- **Decision:** What was chosen and why?
- **Consequences:** What are the implications?

### Step 2: Check for Related Items
Search for related tasks and decisions:
```
DailyPlanner-search_tasks(query: "[decision topic]")
notion-API-post-search(query: "[decision topic]")
```

### Step 3: Create ADR in Notion

#### Resolve Parent Page
Before creating the ADR page, find the Architecture Decisions parent page in Notion:
```
notion-API-post-search(query: "Architecture Decisions", filter: { property: "object", value: "page" })
```
Use the first result's `id` as the `parent.page_id`.

**If no result found:**
```
ask_user: "I couldn't find an 'Architecture Decisions' page in Notion. Please provide the Notion page URL, or create one first."
```
Extract the page ID from the URL and use it as the parent.

#### Create ADR Page
```
notion-API-post-page(
  parent: { page_id: "[resolved Architecture Decisions page ID]" },
  properties: { title: [{ text: { content: "ADR-[number]: [Title]" } }] },
  children: [structured ADR content]
)
```

ADR structure:
```markdown
# ADR-[number]: [Title]

**Status:** [Proposed | Accepted | Deprecated | Superseded]
**Date:** [date]
**Decision Makers:** [who was involved]
**Team:** [which team this affects]

## Context
[What is the issue that we're seeing that is motivating this decision?]

## Decision
[What is the change that we're proposing and/or doing?]

## Options Considered

### Option 1: [Name]
- ✅ Pros: [advantages]
- ❌ Cons: [disadvantages]

### Option 2: [Name]
- ✅ Pros: [advantages]
- ❌ Cons: [disadvantages]

### Option 3: [Name]
- ✅ Pros: [advantages]
- ❌ Cons: [disadvantages]

## Consequences
- [What becomes easier or more difficult as a result]
- [Follow-up actions needed]

## Related
- Task: [linked task if any]
- Previous ADR: [if this supersedes another]
- Documentation: [relevant docs]
```

### Step 4: Link to Task
If the ADR relates to a Daily Planner task:
```
DailyPlanner-add_activity_log(
  taskId: "[task_id]",
  description: "Architecture Decision Record created: ADR-[number] — [title]. Documented in Notion."
)
```

### Step 5: Summary
```markdown
## ✅ ADR Created

- **ADR:** ADR-[number]: [Title]
- **Status:** [status]
- **Saved to:** Notion — [page title]
- **Team:** [affected team]
- **Linked task:** [task if any]
```

## Tools & APIs Used
- `notion-API-post-page` — Create ADR page
- `notion-API-post-search` — Find related decisions
- `DailyPlanner-search_tasks` — Related tasks
- `DailyPlanner-add_activity_log` — Link to tasks
- `ask_user` — Gather decision details

## Output Format
Structured ADR in Notion with status, context, options, decision, and consequences.

## Notes
- ADRs should be immutable once accepted — use "Superseded by ADR-X" rather than editing
- Number ADRs sequentially for easy reference
- Keep context section factual, not opinion-based
- Consequences should include both positive and negative impacts
