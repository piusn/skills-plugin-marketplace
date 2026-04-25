---
description: "Take, organize, and retrieve study notes from Notion. Use this skill when the user says 'learning notes', 'study notes', 'add notes for [topic]', 'my notes on [subject]', 'review notes', 'what did I learn about', 'show notes', or 'note this down'. Manages structured notes in Notion organized by subject and topic."
---

# Learning Notes Skill

## Context
Study notes live in Notion, organized by subject → topic, with structured sections for different types of notes (concepts, code, questions). This skill handles creating, appending, retrieving, and organizing notes.

## When to Use
- During a study session to capture notes
- After a session to add reflections
- When reviewing what was learned on a topic
- When searching for previously captured concepts or code

## Workflow

### Mode 1: Add Notes

#### Step 1: Identify the Topic
If the user specifies a topic:
```
notion-API-post-search(query: "[topic name]", filter: { property: "object", value: "page" })
```

If not specified, ask:
```
ask_user: "Which topic are these notes for?"
  choices: [list from DailyPlanner-get_topics or recent subjects]
```

#### Step 2: Determine Note Type
```
ask_user: "What type of note?"
  choices: ["📝 Study notes", "💡 Key concept", "💻 Code snippet", "❓ Question/gap", "📊 Summary"]
```

#### Step 3: Capture and Save

**Study Notes** → Append under "📝 Study Notes":
```
notion-API-patch-block-children(block_id: "[topic_page_id]", children: [
  { "heading_3": { "rich_text": [{ "text": { "content": "[Date] — [optional subtitle]" } }] } },
  { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[note content]" } }] } }
])
```

**Key Concept** → Append under "💡 Key Concepts":
```json
[
  { "callout": {
    "rich_text": [{ "text": { "content": "[concept name]: [explanation]" } }],
    "icon": { "emoji": "💡" }
  }}
]
```

**Code Snippet** → Append under "💻 Code Snippets":
```json
[
  { "paragraph": { "rich_text": [
    { "text": { "content": "[description]" }, "annotations": { "bold": true } }
  ]}},
  { "code": {
    "rich_text": [{ "text": { "content": "[code]" } }],
    "language": "[language]"
  }}
]
```

**Question** → Append under "❓ Questions & Gaps":
```json
[
  { "to_do": {
    "rich_text": [{ "text": { "content": "[question]" } }],
    "checked": false
  }}
]
```

**Summary** → Append under "📝 Study Notes" as a callout:
```json
[
  { "callout": {
    "rich_text": [{ "text": { "content": "📊 Summary — [Date]\n[summary content]" } }],
    "icon": { "emoji": "📊" }
  }}
]
```

---

### Mode 2: Retrieve Notes

#### Step 1: Find the Topic Page
```
notion-API-post-search(query: "[topic or subject name]")
```

#### Step 2: Load Notes
```
notion-API-get-block-children(block_id: "[topic_page_id]")
```

#### Step 3: Present Notes
Format the retrieved Notion blocks into readable markdown:

```markdown
## 📖 Notes: [Topic Name]
**Subject:** [Subject] | **Last updated:** [date]

### 💡 Key Concepts
- **[Concept 1]:** [explanation]
- **[Concept 2]:** [explanation]

### 💻 Code Snippets
**[Description]:**
```[language]
[code]
```

### ❓ Open Questions
- [ ] [Question 1]
- [ ] [Question 2]
- [x] [Answered question]

### 📝 Recent Notes
**[Date]:**
- [note 1]
- [note 2]
```

---

### Mode 3: Search Across Notes

When the user asks "what did I learn about [concept]":

#### Step 1: Search Notion
```
notion-API-post-search(query: "[concept]", filter: { property: "object", value: "page" })
```

#### Step 2: Check Multiple Pages
For each matching page, scan content:
```
notion-API-get-block-children(block_id: "[page_id]")
```

#### Step 3: Present Findings
```markdown
## 🔍 Notes mentioning "[concept]"

### From: [Topic 1] (Subject: [Subject])
- [relevant notes]

### From: [Topic 2] (Subject: [Subject])
- [relevant notes]
```

---

### Mode 4: Organize & Clean Up

Periodically review and organize notes:

1. **Move answered questions** — Check to_do items and mark resolved ones
2. **Create summary** — Compile key concepts into a topic summary
3. **Cross-reference** — Link related concepts across topics
4. **Archive completed** — Move completed topic notes to a "Completed" section

## Tools & APIs Used
- `notion-API-post-search` — Find topic/subject pages
- `notion-API-get-block-children` — Read existing notes
- `notion-API-patch-block-children` — Append new notes
- `notion-API-update-a-block` — Update existing blocks (e.g., check to_do)
- `DailyPlanner-get_topics` — List available topics
- `DailyPlanner-get_subjects` — List subjects
- `ask_user` — Topic selection, note type, content capture

## Output Format
Notes saved confirmation with preview, or retrieved notes formatted as readable markdown.

## Notion Block Conventions
| Note Type | Notion Block | Section |
|-----------|-------------|---------|
| Study note | `bulleted_list_item` | 📝 Study Notes |
| Key concept | `callout` with 💡 | 💡 Key Concepts |
| Code snippet | `code` block | 💻 Code Snippets |
| Question | `to_do` (unchecked) | ❓ Questions & Gaps |
| Summary | `callout` with 📊 | 📝 Study Notes |
| Session log | `callout` with 📊 | Top of page |

## Notes
- Always date-stamp notes for chronological tracking
- Code snippets must include the language for syntax highlighting
- Questions become the starting point for next study session
- Summaries help with retention — encourage creating one per session
- If no Notion page exists for a topic, suggest running `learning-setup` first
