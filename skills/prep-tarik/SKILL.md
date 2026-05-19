---
description: "Prepare for meetings or emails with Tarik (skip manager). Use this skill when the user says 'prepare for Tarik', 'Tarik meeting', 'skip level prep', 'Tarik update', 'Tarik email', or 'skip manager prep'. Compiles official task impact, blockers, cross-team efforts, and formats for executive communication."
---

# Prep for Tarik (Skip Manager) Skill

## Context
Tarik is Pius's skip manager. Communication with Tarik should be impact-focused: what was accomplished (WHAT), how it was done (HOW), any blockers, and which teams are involved. This aligns with the Microsoft performance review framework (impact-tracker).

## When to Use
- Before 1:1 or group meetings with Tarik
- When composing email updates to Tarik
- When preparing skip-level reviews

## Workflow

### Step 1: Get Official Tasks
Pull all tasks tagged "official" (these are the work items that matter for skip-level):
```
DailyPlanner-get_tasks(tag: "official")
```

Also get completed official tasks for the reporting period:
```
DailyPlanner-get_tasks(tag: "official", status: "Completed")
```

### Step 2: Map to Impact Framework
For each official task, extract:

**WHAT** — Results delivered:
- Measurable outcomes
- Security, quality, and AI contributions
- Link to which goal the task maps to

**HOW** — Behaviors demonstrated:
- Growth mindset and curiosity
- Collaboration across teams
- Driving excellence

Use the `impact-tracker` skill data if available — check for existing impact logs on these tasks.

### Step 3: Identify Blockers & Collaboration
For in-progress and blocked tasks:
- What's blocking progress?
- Which team is the blocker related to?
- What help is needed?

Cross-reference with teams:
```
Use my-teams skill context to map each task to its team
```

### Step 4: Get Cross-Team Context
Use WorkIQ for recent cross-team interactions:
```
workiq-ask_work_iq: "What are the key cross-team discussions or decisions I've been involved in recently with teams under ADC Platform Health?"
```

### Step 5: Compose Update
Format for executive consumption:

```markdown
# Update for Tarik — [Date]

## 🎯 Key Accomplishments
| Task | Team | Impact | Status |
|------|------|--------|--------|
| [Task title] | Reliability DE | [measurable outcome] | ✅ Complete |
| [Task title] | Benchmarking | [outcome] | 🔄 In Progress |

## 💡 How I'm Driving Impact
- **Growth & Curiosity:** [example behavior]
- **Collaboration:** [cross-team example]
- **Quality & AI:** [contribution]

## 🚧 Blockers & Risks
| Blocker | Team | Impact | Help Needed |
|---------|------|--------|-------------|
| [blocker] | [team] | [impact] | [ask] |

## 🤝 Cross-Team Efforts
- [Team A]: [collaboration summary]
- [Team B]: [collaboration summary]

## 📌 Next Period Focus
- [Top priorities for upcoming period]
```

### Step 6: Offer Export Options
Ask if the user wants to:
- Copy to clipboard for email
- Export as Word doc (using `markdown-to-word` skill)
- Save to Notion for reference

## Tools & APIs Used
- `DailyPlanner-get_tasks` — Official tasks (tag: "official")
- `impact-tracker` skill — Impact framing (WHAT/HOW)
- `my-teams` skill — Team mapping
- `workiq-ask_work_iq` — Cross-team context
- `DailyPlanner-get_goals` — Goal alignment
- `markdown-to-word` skill — Optional export
- `ask_user` — Clarify period or add context

## Output Format
Executive-ready update with accomplishments, behaviors, blockers, and cross-team efforts in a clean table format.

## Notes
- Tarik cares about impact and cross-team collaboration — lead with outcomes, not activities
- Always map work to the broader team/org goals
- Blockers should come with a clear ask — what help is needed
- Keep it concise — skip-level updates should be scannable in 2 minutes

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
  description = "Surfaced by: prep-tarik · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "prep-tarik"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.