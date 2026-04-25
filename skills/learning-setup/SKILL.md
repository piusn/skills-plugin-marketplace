---
description: "Set up a new learning path with subjects, topics, resources, and Notion pages. Use this skill when the user says 'new learning path', 'start learning [subject]', 'create learning plan', 'set up subject', 'learning roadmap', 'I want to learn [topic]', or 'create study plan'. Creates the structure in both Daily Planner (progress tracking) and Notion (notes & content)."
---

# Learning Setup Skill

## Context
Learning is tracked across two systems:
- **Daily Planner** — Subjects, topics, resources, and progress percentages
- **Notion** — Study notes, key concepts, code snippets, and rich content per topic

This skill sets up both systems for a new learning path, creating a consistent structure that the `learning-session`, `learning-notes`, and `learning-review` skills rely on.

## Learning Philosophy
This skill is designed for a software architect who needs **breadth and depth** across technologies:

- **Fundamentals first** — Never learn a technology in isolation. Start with the underlying principles (e.g., learn distributed systems concepts before Kubernetes)
- **Hands-on always** — Every topic must have practical exercises, runbooks, and step-by-step projects
- **Interconnected learning** — Technologies don't exist in silos. Map connections between subjects (e.g., event-driven architecture connects to Kafka, CQRS, and microservices)
- **Revision by design** — Spaced repetition ensures knowledge sticks. Every topic gets a revision schedule
- **Team-aligned** — Prioritize technologies your teams are actively using or evaluating
- **Architect's lens** — Learn both the "how" (hands-on) and the "why" (architecture decisions, trade-offs, when to use vs. not use)

## When to Use
- When the user wants to learn something new
- When structuring a learning path for an existing subject
- When adding topics and resources to a subject

## Workflow

### Step 1: Define the Learning Path
Ask the user what they want to learn:
```
ask_user: "What do you want to learn? Provide the subject and any specific topics you have in mind."
```

### Step 2: Check Existing Subjects
Search Daily Planner for existing subjects:
```
DailyPlanner-get_subjects(status: "Active")
```

If the subject already exists, offer to add topics to it rather than creating a duplicate.

### Step 3: Research & Structure the Path
Use web search to build a comprehensive learning roadmap:
```
web_search: "[subject] learning path roadmap for intermediate developers"
```

Structure the path into:
- **Subject** — The overarching area (e.g., "Kubernetes")
- **Topics** — Logical learning units (e.g., "Pods & Containers", "Services & Networking")
- **Resources** — Specific materials per topic (videos, articles, courses, books)

#### For Each Topic, Include:

1. **Fundamentals section:**
   - Core concepts and principles (not tool-specific)
   - Recommended books or foundational resources
   - "Explain it like I'm 5" summary

2. **Time-blocked learning units:**
   Break each topic into small, focused learning blocks:
   ```markdown
   ### Learning Blocks
   | # | Block Title | Duration | Type | Definition of Done |
   |---|------------|----------|------|-------------------|
   | 1 | Core concepts & terminology | 30 min | 📖 Read + Notes | Can explain 5 key concepts without reference |
   | 2 | Official quickstart tutorial | 45 min | 🔧 Hands-on | Tutorial complete, running locally |
   | 3 | Architecture patterns | 30 min | 📖 Read + Diagram | Mermaid diagram of key patterns drawn |
   | 4 | Build mini-project part 1 | 45 min | 🛠️ Project | API endpoint working with tests |
   | 5 | Build mini-project part 2 | 45 min | 🛠️ Project | Full feature complete, committed |
   | 6 | Trade-offs & when NOT to use | 25 min | 🤔 Analysis | Written comparison with alternatives |
   | 7 | Review & teach-back | 25 min | 📝 Consolidate | Summary doc written, questions answered |
   ```

   **Block design rules:**
   - Each block is **25-45 minutes** (one focused session)
   - Every block has a **clear Definition of Done** — measurable, verifiable
   - Blocks build on each other but can be paused between
   - Mix theory (📖) and practice (🔧🛠️) blocks — never more than 2 theory blocks in a row
   - Final block should always be consolidation/teach-back

3. **Hands-on exercises:**
   - Step-by-step tutorial or lab
   - Runbook: a procedure to follow and practice
   - Workbook: exercises with increasing difficulty

4. **Mini-project:**
   - A small, self-contained project that demonstrates the topic
   - Should take 2-4 hours to complete (broken into blocks above)
   - Must produce a working artifact (code, config, deployment)
   - **Definition of Done:** working code committed, tests passing, README written

4. **Cross-topic connections:**
   Map how this topic connects to other subjects:
   ```markdown
   ### Connections
   - **Prerequisite for:** [topics that build on this]
   - **Related to:** [topics in other subjects that complement this]
   - **Combine with:** [topics for a cross-cutting project]
   ```

5. **Curated resources:**
   ```markdown
   ### Resources
   | Type | Title | Link | Notes |
   |------|-------|------|-------|
   | 📖 Book | [title] | [link] | Chapters [X-Y] relevant |
   | 📝 Blog | [title] | [link] | Excellent practical guide |
   | 🎥 Video | [title] | [link] | [duration] |
   | 💻 GitHub Repo | [repo] | [link] | Study the [component] for patterns |
   | 📚 Course | [title] | [link] | Modules [X-Y] |
   | 🔗 Official Docs | [title] | [link] | Reference |
   ```

6. **Public repos to study:**
   Search for well-architected open-source projects that demonstrate the topic:
   ```
   github-mcp-server-search_repositories: "[topic] example" or "[topic] tutorial"
   ```
   Select repos with good documentation, clear architecture, and active maintenance.

7. **Revision schedule:**
   ```markdown
   ### Revision Plan
   - Day 1: Complete topic
   - Day 3: Quick review (15 min) — revisit key concepts and questions
   - Day 7: Practice exercise — redo the mini-project from memory
   - Day 14: Teach-back — explain the topic as if teaching someone
   - Day 30: Integration project — use this topic with 2+ other topics
   - Day 90: Deep review — read advanced material, update notes
   ```

### Step 4: Discover Team Technologies (for team-aligned learning)
When setting up learning paths, check what technologies your teams are using:

1. **Check team Notion pages:**
   ```
   notion-API-post-search(query: "teams")
   ```
   Look for technology stacks, products, and tools used by each team.

2. **Scan team repos:**
   Use GitHub to check languages, frameworks, and dependencies across team repositories.

3. **Check eng.ms:**
   ```
   enghub-search(query: "[technology] architecture")
   ```
   Find internal documentation, TSGs, and best practices for technologies in use.

4. **Map technology gaps:**
   Compare technologies used by teams against your current learning subjects. Flag gaps:
   ```markdown
   ## Team Technology Alignment
   | Technology | Used By Teams | Your Knowledge | Gap |
   |-----------|--------------|---------------|-----|
   | Kubernetes | Teams A, B, C | 🟡 Intermediate | Deep dive on networking, security |
   | CosmosDB | Teams B, D | 🔴 Beginner | Need fundamentals + data modeling |
   | gRPC | Team A | ❌ Not started | Add to learning path |
   ```

### Step 5: Propose the Learning Plan
Present for approval:

```markdown
# 📚 Learning Plan: [Subject]

## Overview
[Brief description of what will be covered and why]

## Topics & Resources

### Topic 1: [Topic Name]
**Description:** [What this covers]
**Resources:**
1. 📹 [Video title] — [URL] (~[X] min)
2. 📖 [Article title] — [URL]
3. 📚 [Book/Course] — [URL]

### Topic 2: [Topic Name]
**Description:** [What this covers]
**Resources:**
1. 📹 [Video] — [URL]
2. 📖 [Article] — [URL]

### Topic 3: [Topic Name]
...

## Suggested Order
1. [Topic] → 2. [Topic] → 3. [Topic] → ...

## Estimated Duration
- Total topics: [X]
- Estimated hours: [X]
- Suggested pace: [X topics/week]
```

```
ask_user: "Does this learning plan look good? Want to add, remove, or reorder anything?"
```

### Step 6: Create in Daily Planner

**Create subject (if new):**
```
DailyPlanner-create_subject(
  name: "[Subject Name]",
  description: "[Brief description and learning objectives]",
  tags: "[relevant tags]",
  status: "Active"
)
```

**Create topics (in order):**
For each topic:
```
DailyPlanner-create_topic(
  subjectId: "[subject_id]",
  name: "[Topic Name]",
  description: "[What this topic covers]",
  order: [sequence number],
  status: "Not Started",
  tags: "[relevant tags]"
)
```

**Create resources under each topic:**
For each resource:
```
DailyPlanner-create_resource(
  topicId: "[topic_id]",
  title: "[Resource Title]",
  type: "[Book|Video|Article|Course|Podcast|Other]",
  url: "[URL if available]",
  totalDurationMinutes: [X for videos/courses],
  totalPages: [X for books/articles],
  totalChapters: [X for books],
  totalModules: [X for courses],
  status: "NotStarted"
)
```

### Step 7: Create Notion Structure
Create a Notion page hierarchy for notes:

**Create subject page:**
```
notion-API-post-page(
  parent: { page_id: "[learning parent page id]" },
  properties: { title: [{ text: { content: "📚 [Subject Name]" } }] }
)
```

With initial content blocks:
```json
[
  { "heading_1": { "rich_text": [{ "text": { "content": "📚 [Subject Name]" } }] } },
  { "paragraph": { "rich_text": [{ "text": { "content": "[Description and learning objectives]" } }] } },
  { "divider": {} },
  { "heading_2": { "rich_text": [{ "text": { "content": "🗺️ Learning Roadmap" } }] } },
  { "numbered_list_item": { "rich_text": [{ "text": { "content": "[Topic 1]" } }] } },
  { "numbered_list_item": { "rich_text": [{ "text": { "content": "[Topic 2]" } }] } },
  { "divider": {} },
  { "heading_2": { "rich_text": [{ "text": { "content": "📝 Quick Reference" } }] } },
  { "paragraph": { "rich_text": [{ "text": { "content": "Key concepts and cheat sheets go here." } }] } }
]
```

**Create topic pages (as children of subject page):**
For each topic:
```
notion-API-post-page(
  parent: { page_id: "[subject_page_id]" },
  properties: { title: [{ text: { content: "📖 [Topic Name]" } }] }
)
```

With template content:
```json
[
  { "heading_1": { "rich_text": [{ "text": { "content": "[Topic Name]" } }] } },
  { "callout": { "rich_text": [{ "text": { "content": "Daily Planner Topic ID: [topic_id]" } }], "icon": { "emoji": "🔗" } } },
  { "divider": {} },
  { "heading_2": { "rich_text": [{ "text": { "content": "📝 Study Notes" } }] } },
  { "paragraph": { "rich_text": [{ "text": { "content": "Notes from study sessions will be added here." } }] } },
  { "divider": {} },
  { "heading_2": { "rich_text": [{ "text": { "content": "💡 Key Concepts" } }] } },
  { "paragraph": { "rich_text": [{ "text": { "content": "Important concepts, definitions, and mental models." } }] } },
  { "divider": {} },
  { "heading_2": { "rich_text": [{ "text": { "content": "💻 Code Snippets" } }] } },
  { "paragraph": { "rich_text": [{ "text": { "content": "Useful code examples and patterns." } }] } },
  { "divider": {} },
  { "heading_2": { "rich_text": [{ "text": { "content": "📚 Resources" } }] } },
  { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[Resource 1 with link]" } }] } },
  { "divider": {} },
  { "heading_2": { "rich_text": [{ "text": { "content": "❓ Questions & Gaps" } }] } },
  { "paragraph": { "rich_text": [{ "text": { "content": "Things to revisit or explore further." } }] } }
]
```

### Step 8: Link the Systems
Store the Notion page IDs as a reference. Add a callout block on each Notion topic page with the Daily Planner topic ID, and log the Notion page ID in the topic description in Daily Planner:
```
DailyPlanner-create_topic (include in description):
  description: "... | Notion Page: [notion_page_id]"
```

### Step 9: Summary

```markdown
## ✅ Learning Path Created: [Subject]

### Daily Planner
- Subject: [name] (ID: [id])
- Topics: [X] created
- Resources: [X] added

### Notion
- Subject page: [title] (ID: [page_id])
- Topic pages: [X] created with note templates

### Next Steps
- Start studying with: `"study [first topic name]"`
- Take notes with: `"learning notes [topic]"`
- Check progress with: `"learning review"`
```

### Step 10: Define Cross-Subject Integration Projects
After creating topics, identify opportunities to combine multiple subjects into integration projects:

1. **Scan for natural combinations:**
   Look across all learning subjects for topics that naturally work together:
   ```markdown
   ## Integration Projects
   | Project | Topics Combined | Complexity | Status |
   |---------|----------------|-----------|--------|
   | Build event-driven microservice | Kafka + CQRS + Docker + gRPC | Advanced | Not started |
   | Deploy ML model to production | Python ML + Docker + Kubernetes + CI/CD | Advanced | Not started |
   | Full-stack app with auth | React + Node.js + OAuth2 + PostgreSQL | Intermediate | Not started |
   ```

2. **Create project briefs:**
   For each integration project, create a brief in Notion:
   - What you'll build
   - Which topics it reinforces
   - Estimated time
   - Success criteria
   - Public repos to reference

3. **Schedule integration projects:**
   Place integration projects after their component topics are at 50%+ progress.

## Tools & APIs Used
- `DailyPlanner-create_subject` — Create subject
- `DailyPlanner-create_topic` — Create topics
- `DailyPlanner-create_resource` — Add resources
- `DailyPlanner-get_subjects` — Check existing
- `notion-API-post-page` — Create Notion pages
- `notion-API-patch-block-children` — Add content blocks
- `notion-API-post-search` — Find team pages and existing content
- `web_search` — Research learning paths and technology trends
- `github-mcp-server-search_repositories` — Find public repos to study
- `enghub-search` — Find internal docs, TSGs, and best practices
- `ask_user` — Confirm plan and gather input

## Output Format
Structured learning plan → creation summary with IDs from both systems → next steps.

## Notion Page Structure Convention
```
📚 [Subject Name]              ← Subject page
  ├── 📖 [Topic 1]             ← Topic page with note template
  ├── 📖 [Topic 2]
  ├── 📖 [Topic 3]
  └── 📝 Quick Reference       ← Cheat sheet / summary page
```

## Proactive Technology Discovery
Periodically (weekly or during `learning-review`), scan for new technologies to learn:

1. **Team technology changes:**
   Check if teams have adopted new tools, frameworks, or patterns.

2. **Industry trends:**
   ```
   web_search: "trending software engineering technologies [current year]"
   web_search: "new [cloud/AI/ML/data] technologies for architects"
   ```

3. **Adjacent technologies:**
   For each subject you're learning, identify adjacent technologies:
   - If learning Kubernetes → consider: Istio, Helm, ArgoCD, Crossplane
   - If learning Python ML → consider: MLflow, Ray, LangChain, vector databases
   - If learning Azure → consider: Bicep, Azure Container Apps, Azure AI Studio

4. **Propose to user:**
   ```
   🆕 Technology suggestions based on your learning and team needs:
   | Technology | Why Learn It | Connects To | Priority |
   |-----------|-------------|------------|----------|
   | [tech] | [rationale] | [existing subjects] | [High/Medium/Low] |
   ```
   Ask: "Would you like to add any of these to your learning path?"

## Notes
- Always check for existing subjects before creating duplicates
- Topic order matters — set the `order` field for recommended learning sequence
- Resources should include estimated time so learners can plan sessions
- The Notion template structure ensures consistent note-taking across all topics
- Store cross-references (Daily Planner ID ↔ Notion page ID) in both systems
- Each topic should include fundamentals, hands-on exercises, a mini-project, and a revision schedule
- Map cross-topic connections to enable integration projects
- Periodically discover new technologies via team changes, industry trends, and adjacent tech
