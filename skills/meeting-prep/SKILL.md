---
description: "Prepare for an upcoming meeting with context and talking points. Use this skill when the user says 'prepare for meeting', 'meeting prep', 'get ready for [meeting]', 'prep for [meeting name]', 'what do I need for [meeting]', or 'meeting context'. Pulls previous notes from Notion, pending action items, and relevant communications."
---

# Meeting Prep Skill

## Context
Thorough meeting preparation means walking in with full context: what was discussed before, what's pending, what attendees have been working on, and what questions need answers. This skill builds a comprehensive prep brief by pulling data from Notion (meeting history), Daily Planner (action items), WorkIQ (emails, chats, calendar, documents), and GitHub (PRs, code activity from attendees).

## When to Use
- Before any meeting
- When invoked by `start-day` skill for the day's meetings
- When the user wants context for a specific meeting

## Workflow

### Step 1: Identify the Meeting
If the user specifies a meeting name, search for it:
```
DailyPlanner-get_todays_meetings()
```
Match by title (fuzzy match). If ambiguous, use `ask_user` to clarify which meeting.

If no specific meeting, offer a list of today's meetings to choose from.

### Step 2: Check for Previous Sessions

#### Recurring Meeting Detection
Check if this is a recurring meeting (1:1s, standups, team syncs, weekly reviews):
- Look at the meeting title for patterns: "1:1", "standup", "sync", "weekly", "bi-weekly", "recurring"
- Check if Notion has multiple entries for this meeting title

If this is a recurring meeting, apply these enhanced behaviors:
- **Carry forward agenda:** Pull unresolved items from the last session as top agenda items
- **Track cadence:** Note when the last occurrence was and highlight any skipped sessions
- **Pattern detection:** If the same action item appears in 3+ consecutive meetings, flag it:
  > "⚠️ Action item '[item]' has been carried forward for {N} meetings. Consider escalating or removing."
- **1:1 special handling:** For 1:1 meetings, also pull:
  - Recent team feedback notes (via `team-feedback` skill references)
  - Career/growth discussion points from previous 1:1s
  - Any "parking lot" items from previous sessions

Search Notion for previous meeting notes:
```
notion-API-post-search(query: "[meeting title]", filter: { property: "object", value: "page" })
```

If found, retrieve the latest notes:
```
notion-API-get-block-children(block_id: "[page_id]")
```

Extract:
- Key decisions from last meeting
- Open action items
- Discussion points carried forward

### Step 3: Check Daily Planner Action Items
Search for tasks linked to this meeting or tagged with related topics:
```
DailyPlanner-search_tasks(query: "[meeting title or related keywords]")
```

Also check for tasks with meeting-related tags.

### Step 4: Pull Communication Context
Use WorkIQ to gather rich context from multiple angles:

1. **Recent emails and chats:**
   ```
   workiq-ask_work_iq: "What recent emails or Teams messages are related to [meeting topic/attendees]? Summarize the key points from the last week."
   ```

2. **Meeting agenda and invite details:**
   ```
   workiq-ask_work_iq: "What is the agenda or description for the meeting '[meeting title]' on [date]? Include any attached documents or links."
   ```

3. **Documents shared by attendees:**
   ```
   workiq-ask_work_iq: "What documents, presentations, or files have [attendee names] shared with me recently? List with dates and brief descriptions."
   ```

4. **Previous meeting recordings/transcripts:**
   ```
   workiq-ask_work_iq: "Are there any recordings or transcripts from previous '[meeting title]' meetings?"
   ```

### Step 5: Pull Engineering Context (for technical meetings)
If the meeting involves engineering teams or technical topics, gather code activity:

1. **Recent PRs from attendees:**
   Use GitHub MCP or az CLI to check recent pull requests:
   ```
   github-mcp-server-search_pull_requests: query related to attendees or repos discussed in this meeting context
   ```
   Summarize: what PRs are open, recently merged, or blocked.

2. **Recent commits and code changes:**
   Check for significant recent changes in relevant repos that might be discussion topics.

3. **Open issues:**
   ```
   github-mcp-server-search_issues: query for open issues in repos relevant to the meeting
   ```

4. **Build/pipeline status:**
   Check if there are failing builds or blocked deployments that should be raised.

**Skip this step** for non-technical meetings (1:1s focused on career, admin meetings, etc.).

### Step 6: Build Prep Brief
Compose a comprehensive preparation document:

```markdown
# 📋 Meeting Prep: [Meeting Title]
**Date:** [date/time] | **Duration:** [X min] | **Location:** [location/link]
**Attendees:** [list] | **Organizer:** [name]
**Next Occurrence:** [date, if recurring]

## 🎯 Meeting Goal
[Pulled from calendar invite description. If none: "⚠️ No agenda provided. Suggested talking points based on previous notes and open action items:"]

## 📝 Previous Meeting Summary ([date of last])
- [Key decisions and outcomes from last meeting]
- [Unresolved items carried forward]

## ✅ Pending Action Items
| Item | Owner | Due | Status |
|------|-------|-----|--------|
| Complete API design | Me | Mar 15 | In Progress |
| Review PR #123 | John | Mar 12 | ⚠️ Overdue |

## 📤 Waiting On Others
| Item | Owner | Due | Meeting Context |
|------|-------|-----|----------------|
| Send specs | John | Mar 14 | Last sync |

## 📬 Recent Communications
- [Relevant emails/chats summary with dates and senders]

## 📄 Shared Documents & Materials
- [Documents shared by attendees — title, date, brief description]
- [Links to relevant specs, designs, or presentations]

## 🔧 Engineering Activity (if technical meeting)
### Recent PRs
| PR | Author | Status | Repo |
|----|--------|--------|------|
| #456 Add caching layer | @John | Open — needs review | service-api |
| #452 Fix timeout bug | @Sarah | Merged 2 days ago | service-api |

### Open Issues
- [Relevant open issues that may need discussion]

### Build/Pipeline Status
- [Any failing builds or blocked deployments]

## ❓ Prepared Questions & Clarifications
Based on the context above, consider asking:
1. [Question derived from overdue action items — "What's blocking X?"]
2. [Question derived from recent comms — "You mentioned Y in your email, can we discuss?"]
3. [Question derived from PR activity — "I saw PR #456, how does this relate to our design?"]
4. [Clarification needed from previous meeting — "Last time we decided Z, are we still aligned?"]
5. [Question about shared documents — "I reviewed the spec you shared, question about section N"]

## 💡 Suggested Talking Points
1. Follow up on [pending/overdue items]
2. Discuss [topic from recent comms]
3. Review [recent engineering changes]
4. [Any blockers to raise]
5. [Parking lot items from previous meetings]
```

### Step 7: Back-to-Back Meeting Check
Check the calendar for the next meeting after this one:
- If the next meeting starts within **30 minutes** of this one ending, add a warning to the prep brief:
  ```
  ⚡ Next meeting: [Title] starts in [X] min after this one ends.
  ```
- If prep is being done with **less than 5 minutes** before the meeting starts, use **fast-path mode**:
  - Show only: Previous decisions, overdue action items, attendees
  - Skip: Communications search, suggested talking points
  - Note: "⏱️ Fast prep — limited time. Full context available via `meeting prep [title]` after."

### Step 8: Offer to Create Meeting Session
Ask if the user wants to create a dedicated session for this meeting (for note-taking during the meeting).

### Step 9: Live Note-Taking Mode

After presenting the prep brief, if the meeting is about to start or is in progress, offer to switch to live note-taking mode:

```
📝 Ready for live notes? I'll capture as we go. Just tell me:
- Decisions made
- Action items (who, what, when)
- Key discussion points
- Follow-ups for next meeting

Say "meeting ended" or "wrap up" when done — I'll invoke the meeting-end skill automatically.
```

#### During the Meeting
While in live note-taking mode:

1. **Capture continuously:** As the user shares information, organize it into:
   - **Decisions:** Prefix with ✅
   - **Action items:** Prefix with 📌 — always capture owner and due date
   - **Discussion points:** Prefix with 💬
   - **Parking lot:** Prefix with 🅿️ for items to revisit later

2. **Command-based capture:** Use quick commands for structured input:
   - `d [text]` — capture a **decision**
   - `a [item] @[owner] [date]` — capture an **action item**  
   - `p [text]` — add to **parking lot** (items to revisit later)
   - `s` — show **running summary**
   - `undo` — remove the last captured item
   - Any other text — captured as a **discussion point**

3. **Decision-triggered prompts:** After each decision is captured, ask:
   > "Any action items from this decision?"

4. **Handoff to meeting-end:** When the user says "meeting ended", "wrap up", or "done", automatically invoke the `meeting-end` skill with the captured notes pre-loaded — no need for the user to re-enter them.

## Tools & APIs Used
- `DailyPlanner-get_todays_meetings` — Find the meeting
- `DailyPlanner-get_meeting` — Get meeting details
- `notion-API-post-search` — Find previous meeting notes in Notion
- `notion-API-get-block-children` — Read previous meeting notes
- `DailyPlanner-search_tasks` — Related action items
- `workiq-ask_work_iq` — Recent communications, meeting agenda, shared documents, recordings
- `github-mcp-server-search_pull_requests` — Recent PRs from attendees
- `github-mcp-server-search_issues` — Open issues in relevant repos
- `github-mcp-server-list_pull_requests` — PR status in specific repos
- `ask_user` — Clarify meeting selection

## Output Format
Structured prep brief with previous context, pending items, communications summary, and suggested talking points.

## Notes
- If no previous notes exist in Notion, note this and suggest starting documentation
- For recurring meetings (standups, 1:1s), the previous session is especially important
- Keep the prep brief scannable — the user needs to read it quickly before the meeting starts
- Live note-taking mode feeds directly into `meeting-end` — no duplicate data entry needed
- For important meetings, suggest recording key decisions verbatim rather than summarizing
