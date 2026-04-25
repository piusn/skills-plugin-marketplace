---
description: "Track and manage PR reviews across teams. Use this skill when the user says 'PR reviews', 'review queue', 'pending reviews', 'what PRs need review', 'review status', or 'code reviews'. Shows pending reviews prioritized by team and age."
---

# PR Review Coordinator Skill

## Context
Working across 8 teams means many pull requests to review. This skill helps track pending reviews, prioritize them, and maintain review cadence across all teams.

## When to Use
- When checking what PRs need review
- During start-day to see review queue
- Weekly review of PR review cadence
- When a team asks about review status

## Workflow

### Step 1: Check Pending Reviews
Use WorkIQ to get PR review requests:
```
workiq-ask_work_iq: "What pull requests or code reviews are assigned to me or awaiting my review? List them with repository, PR number, author, title, and how long they've been waiting."
```

### Step 2: Check GitHub Directly
For key repositories, also check GitHub:
```
github-mcp-server-search_pull_requests(
  query: "review-requested:@me state:open",
  sort: "created",
  order: "asc"
)
```

### Step 3: Organize by Team
Map repositories to teams using `my-teams` skill context:

```markdown
# 📋 PR Review Queue

## Summary
- 🔴 Overdue (>3 days): [X] PRs
- 🟡 Aging (1-3 days): [X] PRs
- 🟢 Fresh (<1 day): [X] PRs

## By Team

### Reliability Data Engineering
| PR | Repo | Author | Title | Age | Priority |
|----|------|--------|-------|-----|----------|
| #123 | FastFiler | @dev1 | Fix crash handling | 🔴 5 days | High |
| #456 | BangAnalyze | @dev2 | Add new filter | 🟢 4 hrs | Normal |

### Benchmarking
| PR | Repo | Author | Title | Age | Priority |
|----|------|--------|-------|-----|----------|
| #789 | FunGates | @dev3 | Update scoring | 🟡 2 days | Medium |
```

### Step 4: Review Cadence Report (Weekly)
Show review metrics:

```markdown
## 📊 Review Cadence (This Week)
| Metric | Value |
|--------|-------|
| PRs Reviewed | [X] |
| Avg Review Time | [X] hours |
| Teams Covered | [X] of 8 |
| Overdue Reviews | [X] |

### Teams Not Reviewed This Week
- [Team with no reviews]
```

### Step 5: Prioritize
Suggest review order based on:
1. Age (oldest first)
2. Team coverage (teams with no recent reviews)
3. PR size (smaller PRs first for quick wins)
4. Author (direct reports first)

## Tools & APIs Used
- `workiq-ask_work_iq` — PR review notifications
- `github-mcp-server-search_pull_requests` — GitHub PR search
- `github-mcp-server-pull_request_read` — PR details
- `my-teams` skill — Team mapping

## Output Format
Prioritized review queue organized by team with age indicators and cadence metrics.

## Notes
- PRs older than 3 days should be flagged as overdue
- Aim for review within 24 hours for team PRs
- Track review cadence to ensure all teams get attention
- Small PRs (<100 lines) should be reviewed first for quick turnaround
