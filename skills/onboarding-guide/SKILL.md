---
description: "Generate team-specific onboarding guides for new team members. Use this skill when the user says 'create onboarding', 'onboarding guide', 'new team member', 'new hire guide', 'onboarding for [team]', or 'welcome guide'. Produces comprehensive onboarding documentation from Notion, codebase, and existing docs."
---

# Onboarding Guide Skill

## Context
Each of the 8 teams under Pius's architecture role may onboard new engineers. A standardized onboarding guide ensures consistency while being customized per team's tools, repos, and workflows.

## When to Use
- When a new team member is joining
- When updating onboarding materials
- When standardizing onboarding across teams

## Workflow

### Step 1: Identify the Team
```
ask_user: "Which team is the new member joining?"
  choices: ["Reliability Data Engineering", "Benchmarking", "Data Analytics & Anomaly Detection", "Sustainability", "Power, Performance & Sustainability DE", "Gates & Defense", "COSINE", "Team Duma"]
```

### Step 2: Gather Team Information
Pull team details from Notion:
```
notion-API-get-block-children(block_id: "[team notion page id from my-teams]")
```

Get the team documentation standard:
```
notion-API-get-block-children(block_id: "31c891a6-db0d-8125-98b4-fab2e24f72ff")
```

### Step 3: Gather Codebase Information
Check the team's repositories:
- Look for existing README files
- Check for `.github/copilot-instructions.md`
- Identify build and test instructions
- Find architecture documentation

### Step 4: Check eng.ms for Documentation
```
enghub-search(query: "[team name] onboarding")
enghub-search(query: "[product name] getting started")
```

### Step 5: Compose Onboarding Guide
Create in Notion under the team page:

```markdown
# Welcome to [Team Name]! 🎉

## Week 1: Getting Started

### Day 1-2: Environment Setup
- [ ] Get access to [list of repos]
- [ ] Set up development environment
  - [Specific setup instructions]
- [ ] Request access to [Azure resources, dashboards, etc.]
- [ ] Join Teams channels: [list]
- [ ] Meet the team (intro meeting)

### Day 3-5: Codebase Orientation
- [ ] Read the architecture overview: [link]
- [ ] Run the project locally: [build commands]
- [ ] Run tests: [test commands]
- [ ] Review key documentation: [links]

## Week 2: First Contribution

### Suggested Starter Tasks
- [ ] [Easy bug or improvement]
- [ ] [Documentation update]
- [ ] [Small feature]

### Code Review Process
- [How PRs are submitted and reviewed]
- [CI/CD pipeline overview]
- [Coding standards]

## Key Resources

### People
| Role | Name | Contact |
|------|------|---------|
| Manager | [name] | [alias] |
| Architect | Pius Ngugi | pingugi |
| Senior Dev | [name] | [alias] |

### Links
| Resource | URL |
|----------|-----|
| ADO Board | [link] |
| Product Dashboard | [link] |
| Documentation | [link] |
| Team Notion Page | [link] |

### Meetings
| Meeting | Frequency | Day/Time |
|---------|-----------|----------|
| Standup | Daily | [time] |
| Sprint Planning | Bi-weekly | [time] |
| Retrospective | Bi-weekly | [time] |

## Architecture Overview
[High-level architecture of the team's products with Mermaid diagrams]

## FAQ
- Q: [Common question]
- A: [Answer]
```

### Step 6: Save to Notion
```
notion-API-post-page(
  parent: { page_id: "[team page id]" },
  properties: { title: [{ text: { content: "Onboarding Guide — [Team Name]" } }] },
  children: [structured blocks]
)
```

### Step 7: Offer Export
- Export as Word doc for sharing
- Export as PDF
- Keep in Notion as living document

## Tools & APIs Used
- `my-teams` skill — Team information
- `notion-API-*` — Team pages and documentation standard
- `enghub-search` / `enghub-fetch` — Existing documentation
- Explore agent — Codebase analysis
- `markdown-to-word` skill — Export option
- `ask_user` — Team selection

## Output Format
Comprehensive onboarding guide in Notion with checklists, resource tables, and architecture overview.

## Notes
- Onboarding guides are living documents — update when processes change
- Include both technical setup and team culture aspects
- Starter tasks should be genuinely achievable in the first week
- Include the team's specific coding standards and conventions
