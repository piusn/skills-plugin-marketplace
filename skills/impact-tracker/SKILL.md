---
name: impact-tracker
description: >
  Documents work impact for Microsoft performance reviews. Use this skill when
  starting or completing tasks tagged "official", or when generating a
  consolidated impact report for a review period. Tracks WHAT results were
  delivered and HOW behaviors drove excellence across 4 goals.
---

# Impact Tracker — Performance Review Documentation Skill

Capture and consolidate work impact for Microsoft performance reviews. This skill ensures every "official" task is mapped to goals, outcomes are tracked, and review-time consolidation is effortless.

## Context: Performance Review Framework

Microsoft reviews are structured around two dimensions:

- **WHAT** — What results have you delivered for your goals, keeping security, quality, and AI in mind?
- **HOW** — How have your behaviors and actions led you, your team, and Microsoft to excel, grow, and build trust?

### Review Sections
1. **Reflect on the Past**
   - 1.1 What results did you deliver and how did you do it?
   - 1.2 Reflect on a recent setback — what did you learn and how did you grow?
2. **Plan for the Future**
   - 2.1 What are your goals for the upcoming period?
   - 2.2 How will your actions and behaviors help you reach your goals?

---

## Current Goals (Update each review period)

### Goal 1: Operational Excellence & Technical Leadership
**Objective:** Drive engineering excellence and resource optimization to improve efficiency, quality, and scalability.

**Key Results:**
- Lead system design reviews for all feature work, ensuring >90% alignment with technical standards and minimal rework
- Reduce engineering task lead times by 20% and design-related rework by 30%
- Maintain Azure spending within ±5% of targets and achieve 90% adherence to resource management standards
- Consolidate 100% of redundant Azure resources and reduce resource navigation time by 30%

### Goal 2: Collaboration & Team Growth
**Objective:** Foster a culture of collaboration, mentorship, and continuous learning across teams and geographies, while enabling teams to deliver on their commitments.

**Key Results:**
- Facilitate at least 4 cross-team alignment sessions and organize bi-monthly knowledge-sharing workshops
- Launch a structured mentorship program with 80% participation, increase mentorship and code review frequency by 30%
- Step in to provide clarity and build prototypes when needed, ensuring teams deliver without delays
- Achieve >85% positive feedback on collaboration and learning culture, increase team satisfaction by 20%

### Goal 3: Security & Compliance
**Objective:** Strengthen security posture through proactive engagement, automation, and team accountability.

**Key Results:**
- Develop and deploy an automated daily reminder system for open security items, scaling to at least 2 additional teams
- Achieve 100% completion of mandatory security training for self and 95% team compliance via quarterly audits
- Conduct at least 2 security-focused knowledge-sharing sessions and maintain zero overdue security action items

### Goal 4: Technical Skill Development in Debugging, Benchmarking & Architectural Leadership
**Objective:** Strengthen expertise in debugging and benchmarking while enhancing architectural skills to drive clarity, collaboration, and technical excellence across teams.

**Key Results:**
- Complete at least 2 advanced debugging courses and 1 benchmarking workshop within the quarter
- Apply debugging and benchmarking techniques in at least 2 major feature reviews
- Lead 3 architecture review sessions focused on clarity in design decisions and cross-team alignment
- Organize 1 technical deep-dive session on debugging or benchmarking best practices
- Document and publish an internal guide on architectural principles and performance optimization

### Core Behaviors (Cross-Cutting)
- **Grow with curiosity and adaptability** — Stay informed on emerging technologies, security practices, and engineering standards
- **Collaborative and inclusive** — Engage team representatives and cross-geo partners in decision-making, mentorship, and knowledge-sharing
- **Be bold, move fast with purpose** — Take ownership, set measurable targets, and lead by example while maintaining quality

---

## Instructions

This skill supports three workflows. Determine which workflow to use based on the user's request.

---

### Workflow 1: Task Start (Impact Planning)

**When to use:** The user is starting work on a task tagged "official", or asks to plan the impact of a task.

**Steps:**

1. **Fetch the task** using `DailyPlanner-get_task` with the provided task ID
2. **Verify** the task has an "Official" tag. If not, inform the user this skill is for official tasks and ask if they want to proceed anyway
3. **Present the 4 goals** and ask the user which goal(s) this task maps to:

   Ask: "Which goal(s) does this task contribute to?"
   - Goal 1: Operational Excellence & Technical Leadership
   - Goal 2: Collaboration & Team Growth
   - Goal 3: Security & Compliance
   - Goal 4: Technical Skill Development

4. **Ask for expected measurable outcome:**
   "What measurable outcome do you expect from this task? (e.g., reduce X by Y%, deliver Z feature, unblock N teams)"

5. **Ask which behaviors will be demonstrated:**
   "Which behaviors will this task demonstrate?"
   - Curiosity & adaptability (learning, exploring new approaches)
   - Collaborative & inclusive (cross-team work, mentorship, knowledge-sharing)
   - Bold, move fast with purpose (ownership, leading by example)

6. **Ask about security/quality/AI relevance (optional):**
   "Does this task contribute to security, quality, or AI? If yes, briefly describe how."

7. **Compose the impact alignment summary** in this format:

   ```
   📊 IMPACT ALIGNMENT
   ═══════════════════
   🎯 Goal(s): [Goal number(s) and name(s)]
   📈 Expected Outcome: [User's response]
   🔑 Key Result(s) Targeted: [Specific KR from the goal]
   💡 Behaviors: [Selected behaviors]
   🔒 Security/Quality/AI: [User's response or "N/A"]
   ```

8. **Log the impact plan** using `DailyPlanner-add_activity_log` with the task ID and the formatted impact alignment as the description
9. **Start the task** using `DailyPlanner-start_task` with the task ID
10. **Confirm** to the user that the task has been started with impact alignment captured

---

### Workflow 2: Task Completion (Impact Capture)

**When to use:** The user is completing a task tagged "official", or asks to document the impact of completed work.

**Steps:**

1. **Fetch the task** using `DailyPlanner-get_task` with the provided task ID
2. **Review the activity logs** to find any existing impact alignment from Workflow 1 (look for "IMPACT ALIGNMENT" in logs)
3. **Present the original impact plan** (if found) and ask the user to reflect on it
4. **Ask for actual measurable outcomes:**
   "What measurable outcomes did you deliver? Be specific with numbers, percentages, or concrete results."

5. **Ask about contributions to security, quality, and/or AI:**
   "How did this work contribute to security, quality, or AI? (Skip if not applicable)"

6. **Ask about behaviors demonstrated:**
   "Which behaviors did you demonstrate during this work?"
   - Growth mindset / curiosity (what did you learn or explore?)
   - Collaboration (who did you work with? how did you enable others?)
   - Boldness / ownership (how did you drive this forward?)

7. **Ask about setbacks and learnings (important for Section 1.2):**
   "Did you encounter any setbacks? What did you learn and how did you grow from them? (This is valuable for your review — even small setbacks count)"

8. **Compose the impact completion summary** in this format:

   ```
   ✅ IMPACT DELIVERED
   ═══════════════════
   🎯 Goal(s): [Goal number(s)]
   📈 Outcomes Delivered: [Actual measurable outcomes]
   🔒 Security/Quality/AI: [Contributions or "N/A"]
   💡 Behaviors Demonstrated:
     - [Behavior]: [Specific example]
   📉 Setbacks & Learnings: [Description or "None"]
   🔑 Key Results Advanced: [Which specific KRs were moved forward]
   ```

9. **Log the impact summary** using `DailyPlanner-add_activity_log` with the task ID, the formatted impact summary, and estimated duration if available
10. **Complete the task** using `DailyPlanner-complete_task` with:
    - `taskId`: the task ID
    - `summary`: A concise summary of what was accomplished and impact delivered
11. **Confirm** to the user with a brief impact summary

---

### Workflow 3: Consolidated Review Report

**When to use:** The user asks to generate a performance review, impact report, review document, or Connect document. Also when they ask "what impact have I had?" or "prepare my review".

**Steps:**

1. **Fetch all official tasks** using `DailyPlanner-get_tasks` with `tag: "official"` and `status: "all"`
2. **For completed and in-progress tasks**, use `DailyPlanner-get_task` to fetch full details including activity logs with impact data. Prioritize tasks that have "IMPACT ALIGNMENT" or "IMPACT DELIVERED" in their activity logs.
3. **Categorize each task** by which goal(s) it maps to (from the impact logs, or infer from task title/description/tags if no impact log exists)
4. **Generate the review report** in this structure:

```markdown
# Performance Review — Impact Report
**Period:** [Infer from task dates or ask user]
**Name:** [Ask user or infer from context]

---

## Section 1: Reflect on the Past

### 1.1 What results did you deliver and how did you do it?

#### Goal 1: Operational Excellence & Technical Leadership
**Results:**
- [Aggregated measurable outcomes from tasks mapped to Goal 1]

**How (Behaviors):**
- [Specific behavior examples from these tasks]

**Security/Quality/AI Contributions:**
- [Relevant contributions from these tasks]

#### Goal 2: Collaboration & Team Growth
[Same structure...]

#### Goal 3: Security & Compliance
[Same structure...]

#### Goal 4: Technical Skill Development
[Same structure...]

---

### 1.2 Reflect on setbacks — what did you learn and how did you grow?

[Aggregate all setback entries from task impact logs]

- **Setback:** [Description]
  **Learning:** [What was learned]
  **Growth:** [How it led to improvement]

---

## Section 2: Plan for the Future

### 2.1 Goals for the upcoming period

[Present the current goals from the embedded goals section above, or ask the user if goals have changed]

### 2.2 How will your actions and behaviors help you reach your goals?

**Grow with curiosity and adaptability:**
- [Planned actions based on learnings from this period]

**Collaborative and inclusive:**
- [Planned collaboration and mentorship activities]

**Be bold, move fast with purpose:**
- [Planned ownership and leadership actions]
```

5. **Ask the user** if they want to:
   - Review and edit the report in the terminal
   - Export to a markdown file
   - Convert to Word document (suggest using `markdown-to-word` skill)

---

## Handling Edge Cases

- **Task without "official" tag:** Inform the user and ask if they want to proceed anyway. If yes, follow the same workflow.
- **Task with no prior impact alignment (Workflow 2):** Skip step 3 and proceed with capturing outcomes directly.
- **No completed tasks for review period:** Generate the report structure with available in-progress tasks and highlight gaps.
- **User provides minimal answers:** Accept brief responses — some documentation is better than none. Encourage specifics but don't block.

## Tips for the Agent

- Be conversational, not bureaucratic — this should feel like a quick reflection, not a form
- Use the user's own words in the impact summaries where possible
- When inferring goal mapping, consider task tags: "Benchmarking" → Goal 4, "Security" → Goal 3, "Engineering Excellence" → Goal 1, team/people tags → Goal 2
- For the review report, prioritize quality over quantity — highlight the top 3-5 impacts per goal rather than listing everything
- Setbacks are valuable — encourage the user to share them as they demonstrate growth mindset

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
  description = "Surfaced by: impact-tracker · What I tried · What was missing · Proposed fix (new tool / field / endpoint / fixed default / doc) · Workaround used (if any)",
  priority    = "P3",          # P2 if it blocks a common workflow; P1 only if it blocks the current request
  type        = "Task",
  tags        = ["mcp-gap", "daily-planner", "impact-tracker"]
)
```

Then acknowledge inline in your reply: `📝 Captured MCP gap: [<id>] <title>`.

- **Do** capture: missing tool, missing field, awkward shape, slow tool, bad default, unclear error, sync mismatch, doc gap.
- **Do NOT** capture: transient network/auth errors, user-data issues, items already in the backlog (search `tags=mcp-gap` first).
- **Never let a gap-capture failure block the user.** If `create_task` itself fails, mention the gap inline so the user can capture it manually.

Full protocol, description template, and examples: [`../_shared/dp-gap-capture.md`](../_shared/dp-gap-capture.md).
The `review-backlog` skill auto-surfaces these items when run from the `daily-planner` repo or any Sokokapu-Limited microservice repo.