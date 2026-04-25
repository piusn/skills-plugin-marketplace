---
description: "Document team member contributions and send appreciation. Use this skill when the user says 'team feedback', 'document team member', 'appreciation', 'team member impact', 'recognize [name]', 'feedback for [name]', or 'people discussion'. Tracks engineer contributions in Notion and helps draft recognition messages."
---

# Team Feedback & Impact Skill

## Context
As the architect working across 8 teams, Pius collaborates with many engineers. This skill helps:
1. Document each engineer's contributions in Notion for people discussions
2. Send daily appreciation messages to high-impact contributors
3. Track feedback coverage to ensure no one is overlooked

## When to Use
- During or after people discussion preparation
- When recognizing a team member's contribution
- Weekly review of team member documentation coverage
- When the user works closely with someone and wants to document impact

## Workflow

### Step 1: Identify Engineers to Document
Check tasks to find collaborators:
```
DailyPlanner-get_tasks(status: "Completed")
DailyPlanner-get_tasks(status: "In Progress")
```

Extract names from task descriptions, action items, and meeting notes.

Cross-reference with teams using `my-teams` skill — get team pages from Notion:
```
notion-API-get-block-children(block_id: "[team notion page id]")
```
Look for "👥 The Team" section to get team members.

### Step 2: Check Existing Documentation
For each identified engineer, search Notion for their page:
```
notion-API-post-search(query: "[engineer name]", filter: { property: "object", value: "page" })
```

### Step 3: Document Contributions
For each engineer with notable impact:

**If page exists** — append new contributions:
```
notion-API-patch-block-children(block_id: "[page_id]", children: [
  { "heading_3": { "rich_text": [{ "text": { "content": "Week of [Date]" } }] } },
  { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[Contribution description]" } }] } },
  { "paragraph": { "rich_text": [{ "text": { "content": "Impact: [how this helped the team/org]" } }] } }
])
```

**If no page exists** — create one:
```
notion-API-post-page(
  parent: { page_id: "[team page id or people parent page]" },
  properties: { title: [{ text: { content: "[Engineer Name] — Feedback & Impact" } }] },
  children: [structured content]
)
```

### Step 4: Draft Appreciation Messages
For engineers with notable impact, draft a short appreciation message:

```markdown
## 🌟 Appreciation Draft for [Name]

Hi [Name],

I wanted to take a moment to recognize [specific contribution]. Your work on [task/feature] has [specific impact — saved time, improved quality, unblocked the team, etc.].

[Specific behavior to call out — collaboration, initiative, quality, etc.]

Thank you for your excellent work! 🙏

— Pius
```

Ask the user to review and approve before sending (this skill drafts, doesn't send).

### Step 5: Coverage Report
Show documentation coverage across teams:

```markdown
## 📋 Team Member Documentation Coverage

### Reliability Data Engineering
| Member | Last Documented | Status |
|--------|----------------|--------|
| [Name 1] | Mar 10, 2026 | ✅ Recent |
| [Name 2] | Feb 15, 2026 | ⚠️ 4 weeks ago |
| [Name 3] | Never | 🔴 Not documented |

### Benchmarking
| Member | Last Documented | Status |
|--------|----------------|--------|
| ... | ... | ... |

### Summary
- ✅ Documented this month: [X] engineers
- ⚠️ Not documented in 4+ weeks: [X] engineers
- 🔴 Never documented: [X] engineers
```

### Step 6: Schedule Reminders
Flag engineers who haven't been documented in over 4 weeks as needing attention.

## Tools & APIs Used
- `DailyPlanner-get_tasks` — Find collaborators from tasks
- `my-teams` skill — Team structure and members
- `notion-API-post-search` — Find existing engineer pages
- `notion-API-post-page` — Create new pages
- `notion-API-patch-block-children` — Append contributions
- `notion-API-get-block-children` — Read team member lists
- `ask_user` — Review appreciation drafts

## Output Format
Notion pages updated, appreciation drafts presented, coverage report table.

## Notes
- Appreciation should be specific — generic "great job" doesn't count
- Focus on behaviors and outcomes, not just activities
- Daily appreciation messages keep team morale high
- Coverage report helps ensure no team member is overlooked in people discussions
- This data feeds directly into people discussion preparation
