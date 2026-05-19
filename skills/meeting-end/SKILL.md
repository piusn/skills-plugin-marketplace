---
description: "Wrap up a meeting by saving notes to Notion and creating action items in Daily Planner. Use this skill when the user says 'meeting ended', 'meeting over', 'wrap up meeting', 'save meeting notes', 'meeting summary', 'close meeting', or 'meeting action items'. Captures notes, decisions, and follow-ups."
---

# Meeting End Skill

## Context
After every meeting, notes should be saved to Notion (the source of truth for meeting history) and action items should be tracked in Daily Planner. This skill ensures nothing from the meeting is lost.

## When to Use
- Immediately after a meeting ends
- When the user wants to document a meeting that already happened
- When reviewing meeting notes that need to be saved

## Workflow

### Step 1: Identify the Meeting
Get the meeting that just ended:
```
DailyPlanner-get_todays_meetings()
```
Match to the most recent meeting (by time), or ask the user to specify if ambiguous.

Get full meeting details:
```
DailyPlanner-get_meeting(id: "[meeting_id]")
```

### Step 2: Capture or Confirm Notes
If the meeting already has notes in Daily Planner (added during the meeting via `DailyPlanner-add_meeting_notes`), use those.

If no notes exist, ask the user:
```
ask_user: "What were the key points from this meeting? Include any decisions made, action items, and follow-ups."
```

#### No-Show / Cancelled Handling
If the meeting had no attendees or was very short with no content:
- Ask: "This meeting appears to have been a no-show or was very brief. Mark as no-show?"
- If confirmed, create a minimal Notion entry:
  ```
  ## YYYY-MM-DD — [Meeting Title] — No-show
  Rescheduled: [Yes/No] | Action: [reschedule action item if needed]
  ```
- Skip Steps 3-9 and proceed to completion.

### Step 3: Structure the Notes
Organize the notes into a structured format:

```markdown
## Meeting: [Title]
**Date:** [date] | **Duration:** [duration] | **Location:** [location]

### Key Decisions
- [Decision 1]
- [Decision 2]

### Discussion Points
- [Topic discussed and outcome]

### Action Items
- [ ] [Action item 1] — Owner: [name] — Due: [date]
- [ ] [Action item 2] — Owner: [name] — Due: [date]

### Waiting On Others
- [ ] [Item assigned to someone else] — Owner: [name] — Due: [date]

### Follow-ups
- [Items to revisit next meeting]
```

### Step 4: Save to Notion
Search for an existing meeting page in Notion (for recurring meetings):
```
notion-API-post-search(query: "[meeting title]", filter: { property: "object", value: "page" })
```

**If existing page found** (recurring meeting):
Append the new notes as a new dated section:
```
notion-API-patch-block-children(block_id: "[page_id]", children: [structured blocks])
```

   **For recurring meetings, also:**
   - Add a separator (divider block) between sessions for clear visual separation
   - Carry forward any unchecked to_do items from the previous session into the new entry
   - If an action item has been carried forward 3+ times, mark it with ⚠️ and suggest escalation

   **Carry-forward tracking:** Append `[×N]` to carried-forward to_do text, incrementing N each session:
   - First carry: `[action item] — @owner — due [date] [×2]`
   - Second carry: `[action item] — @owner — due [date] [×3]`
   - At `[×3]` or higher, add a callout block: `⚠️ Action item carried forward 3+ times. Consider escalating or removing.`
   - Update the page title or a "Last Updated" property if available

**If no page found** (new meeting):
Create a new page under the appropriate parent:
```
notion-API-post-page(
  parent: { page_id: "[meetings parent page]" },
  properties: { title: [{ text: { content: "[Meeting Title]" } }] },
  children: [structured content blocks]
)
```

### Step 5: Create Action Items in Daily Planner
For each action item identified:
```
DailyPlanner-add_meeting_action_item(
  meetingId: "[meeting_id]",
  title: "[action item description]",
  dueDate: "[due date if specified]",
  priority: "[P1-P4 based on urgency]",
  assignee: "[owner if specified]"
)
```

### Step 6: Update Meeting Notes in Daily Planner
Save structured notes back to the Daily Planner meeting:
```
DailyPlanner-add_meeting_notes(
  meetingId: "[meeting_id]",
  notes: "[structured notes summary]"
)
```

### Step 7: Present Summary
Show the user what was captured:

```markdown
## ✅ Meeting Wrapped: [Title]

### Saved To
- 📓 Notion: [page title] — [link if available]
- 📋 Daily Planner: Meeting notes updated

### Action Items Created
| # | Item | Owner | Due | Priority |
|---|------|-------|-----|----------|
| 1 | Review API design | Me | Mar 15 | P2 |
| 2 | Send specs to team | John | Mar 14 | P3 |

### Follow-ups for Next Meeting
- [Items to bring up next time]
```

### Step 8: Schedule Follow-Up Tracking

For each action item created, set up follow-up:

1. **Tag action items with meeting context:**
   Tag each created task with the meeting title so they can be tracked together.

2. **Set follow-up reminders:**
   - If action items have due dates, note them for the user's daily review
   - For items due within 2 days: flag as urgent in the summary
   - For overdue items from previous meetings: highlight them prominently

3. **Create follow-up check:**
   If this is a recurring meeting, add a note to check action item status before the next occurrence:
   ```
   DailyPlanner-add_activity_log(taskId: "[related_task]", log: "Follow up on [meeting] action items before next session on [next date]")
   ```

4. **Suggest follow-up prompt:**
   ```
   📋 Follow-up scheduled. Before the next [meeting title], I'll check:
   - [ ] {Action item 1} — {owner} — due {date}
   - [ ] {Action item 2} — {owner} — due {date}
   
   Use "meeting prep" before the next session to see status.
   ```

### Step 9: Post-Meeting Summary Email

This step can be triggered in two ways:
- **Immediately after a meeting** — as part of the meeting-end flow
- **During daily review** — when catching up on meetings that ended without a summary sent

Ask the user:
```
📧 Would you like me to draft a meeting summary email for the attendees?
```

If yes:

1. **Gather meeting data:**
   - Pull attendee list from meeting details (DailyPlanner or WorkIQ)
   - Pull meeting notes from Notion (the just-saved notes, or retrieve from existing page)
   - Check for any documents, links, or recordings shared during/before the meeting:
     ```
     workiq-ask_work_iq: "Were any files or links shared during the '[meeting title]' meeting on [date]? Include any Teams chat links or documents."
     ```

2. **Draft the summary email:**
   ```markdown
   Subject: Meeting Summary: [Meeting Title] — [Date]

   Hi [attendee first names],

   Here's a summary from our meeting today.

   **Meeting:** [Title]
   **Date:** [date/time] | **Duration:** [X min]
   **Attendees:** [full list]

   ---

   **Meeting Purpose:**
   [Brief 1-2 sentence summary of what the meeting was about — derived from agenda or discussion]

   **Key Decisions:**
   1. [Decision 1 — clear, actionable statement]
   2. [Decision 2]

   **Action Items:**
   | # | Item | Owner | Due Date |
   |---|------|-------|----------|
   | 1 | [Item description] | [Owner] | [Date] |
   | 2 | [Item description] | [Owner] | [Date] |

   **Documents & Links Shared:**
   - 📄 [Document title] — [brief description] — [link]
   - 🔗 [Resource name] — [link]

   **Discussion Highlights:**
   - [Key topic discussed and outcome/conclusion]
   - [Another significant discussion point]

   **Parking Lot (for future discussion):**
   - [Item deferred to next meeting]

   **Follow-ups for Next Meeting:**
   - [Item to revisit]

   ---

   📅 Next meeting: [date/time, if recurring]
   📹 Recording: [link, if available]

   Please confirm your action items by [2 business days from today].
   Reply to this thread with any corrections or additions.

   Best regards,
   [Auto-resolved user name]
   ```

3. **Privacy and sensitivity check:**
   
   ⚠️ **External attendee warning:** If attendees include external domains, warn the user:
   > "This meeting includes external attendees ([domains]). Review the summary for sensitive content before sending."
   
   ⚠️ **Confidential content check:** If meeting notes contain keywords like "confidential", "NDA", "private", "not for distribution":
   > "Meeting notes may contain confidential content. Review carefully before sending."

4. **Present to user for review:**
   Show the draft and ask:
   ```
   📧 Draft ready. Options:
   1. Send as-is
   2. Edit (tell me what to change)
   3. Skip sending
   ```

5. **Send via WorkIQ (if approved):**
   ```
   workiq-ask_work_iq: "Send an email to [attendee list] with subject '[subject]' and body '[formatted body]'"
   ```
   If WorkIQ can't send emails, present the formatted draft for manual copy-paste into Outlook/Teams.

6. **Log that summary was sent:**
   ```
   DailyPlanner-add_activity_log(taskId: "[related_task]", log: "Meeting summary email sent for [meeting title] to [N] attendees")
   ```

## Tools & APIs Used
- `DailyPlanner-get_todays_meetings` — Find the meeting
- `DailyPlanner-get_meeting` — Get meeting details
- `DailyPlanner-add_meeting_notes` — Save notes to Daily Planner
- `DailyPlanner-add_meeting_action_item` — Create tracked action items
- `notion-API-post-search` — Find existing Notion page
- `notion-API-post-page` — Create new Notion page
- `notion-API-patch-block-children` — Append to existing page
- `ask_user` — Capture notes if not already recorded

## Output Format
Summary showing where notes were saved, action items created, and follow-ups noted.

## Notion Block Format for Meeting Notes
When creating Notion blocks, use this structure:
```json
[
  { "divider": {} },
  { "toggle": {
      "rich_text": [{ "text": { "content": "📅 YYYY-MM-DD — [Meeting Title]" } }],
      "children": [
        { "paragraph": { "rich_text": [{ "text": { "content": "Attendees: [names] | Duration: [X min]" } }] } },
        { "heading_3": { "rich_text": [{ "text": { "content": "Key Decisions" } }] } },
        { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[decision]" } }] } },
        { "heading_3": { "rich_text": [{ "text": { "content": "Action Items" } }] } },
        { "to_do": { "rich_text": [{ "text": { "content": "[action] — @[owner] — due [date]" } }], "checked": false } },
        { "heading_3": { "rich_text": [{ "text": { "content": "Waiting On Others" } }] } },
        { "to_do": { "rich_text": [{ "text": { "content": "[item] — @[owner] — due [date]" } }], "checked": false } },
        { "heading_3": { "rich_text": [{ "text": { "content": "Discussion Notes" } }] } },
        { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[topic and outcome]" } }] } },
        { "heading_3": { "rich_text": [{ "text": { "content": "🔜 Parking Lot" } }] } },
        { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[deferred item]" } }] } },
        { "heading_3": { "rich_text": [{ "text": { "content": "Follow-ups" } }] } },
        { "bulleted_list_item": { "rich_text": [{ "text": { "content": "[item for next meeting]" } }] } }
      ]
  }}
]
```

**Scaling note:** Using toggle blocks keeps recurring meeting pages manageable — each session collapses. If a page accumulates 20+ sessions, suggest archiving older entries to a subpage.

## Notes
- Always save to Notion first (source of truth), then Daily Planner (task tracking)
- For recurring meetings, append to the existing Notion page rather than creating new pages
- Action items with no specified due date should default to one week from today
- Tag action items with the relevant team tag when identifiable
- Follow-up tracking integrates with `meeting-prep` — pending action items appear in the next prep brief
- Post-meeting summary emails can be sent immediately or deferred to daily review (e.g., `close-day` or `periodic-review` skills)
- For recurring meetings, follow-up status is automatically checked in the next prep
- For 1:1 meetings, suggest topics the user might want to discuss (career, feedback, blockers) if not already covered
- Recurring meeting notes accumulate on a single Notion page — use toggle blocks for each occurrence
- The summary email should capture the full picture: purpose, decisions, action items, shared docs/links, and next steps

---

## 🔧 MCP/API Gap Capture

This skill interacts with Daily Planner. While using it, **continuously watch
for friction** with the MCP tools or backend APIs — missing tools, missing
fields, awkward multi-call flows, bad defaults, unclear errors, doc gaps —
and capture each one as a backlog item **inline, without blocking the user's
request**:

```
DailyPlanner-create_task(
  title       = "[MCP gap] <short imperative>",
  description = "Surfaced by: meeting-end · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "meeting-end"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.