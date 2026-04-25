---
description: "Sync meetings from Microsoft 365 calendar to Daily Planner. Use this skill when the user says 'sync meetings', 'sync calendar', 'update meetings', 'pull meetings from calendar', or 'check my calendar'. Compares official calendar (via WorkIQ) with Daily Planner and creates/updates meetings accordingly."
---

# Sync Meetings Skill

## Context
Pius uses Microsoft 365 (Outlook/Teams) as his official calendar and Daily Planner for task/meeting management. Meetings should flow from the official calendar into Daily Planner so everything is in one place.

## When to Use
- At the start of each day (invoked by `start-day` skill)
- When the user explicitly asks to sync calendar
- Before meeting preparation to ensure latest meetings are loaded

## Workflow

### Step 1: Pull Official Calendar
Use WorkIQ to get today's meetings (or a specified date):
```
workiq-ask_work_iq: "What meetings do I have today? List all with title, time, duration, location/link, and attendees"
```
If a specific date is needed:
```
workiq-ask_work_iq: "What meetings do I have on [date]? List all with title, time, duration, location/link, and attendees"
```

### Step 2: Pull Daily Planner Meetings
Use the Daily Planner API:
```
DailyPlanner-get_todays_meetings (or with specific date)
```

### Step 3: Compare and Sync
For each meeting from WorkIQ:
1. Check if it already exists in Daily Planner (match by title and approximate time)
2. If **missing**: Create it using `DailyPlanner-create_meeting` with:
   - title, date/time, duration, location/link
   - Tags: relevant team tag if identifiable
3. If **exists but changed**: Update it using `DailyPlanner-update_meeting`:
   - Time changes, location updates, attendee changes
4. If **already synced**: Skip

For each meeting in Daily Planner that is NOT in WorkIQ:
5. If **cancelled/deleted from calendar**: Mark as cancelled:
   ```
   DailyPlanner-update_meeting(id: "[meeting_id]", status: "cancelled")
   ```
   Note: Do NOT auto-delete — the user may have added notes to it.

### Step 4: Report Results
Present a sync summary table:

| Status | Meeting | Time | Action |
|--------|---------|------|--------|
| ✅ New | Meeting Title | 10:00 AM | Created in Daily Planner |
| 🔄 Updated | Meeting Title | 2:00 PM | Time changed (was 1:00 PM) |
| ❌ Cancelled | Meeting Title | 3:00 PM | Marked cancelled in Daily Planner |
| ⏭️ Synced | Meeting Title | 11:00 AM | No changes |

## Tools & APIs Used
- `workiq-ask_work_iq` — Pull meetings from Microsoft 365
- `DailyPlanner-get_todays_meetings` — Get existing Daily Planner meetings
- `DailyPlanner-create_meeting` — Create new meetings

## Output Format
Markdown table showing sync results with status indicators (✅ New, 🔄 Updated, ⏭️ Already synced).

## Graceful Fallback
- If WorkIQ is unavailable, inform the user and suggest checking the calendar manually
- If DailyPlanner is unavailable, show the WorkIQ calendar data directly without syncing
- If matching is ambiguous (multiple meetings with similar titles), ask the user to confirm

## Notes
- Meeting notes are NOT synced here — they are managed by `meeting-prep` and `meeting-end` skills
- Recurring meetings should be created with appropriate recurrence settings
- If WorkIQ returns meetings that look like focus time or OOF blocks, skip them unless the user wants them synced
- Cancelled meetings are marked, not deleted — preserving any notes already attached
- If WorkIQ returns focus time or OOF blocks, skip them unless the user explicitly asks
- For reliable matching, prefer meeting IDs when available over title-based fuzzy matching
