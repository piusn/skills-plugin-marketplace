# Daily Planner MCP / API Gap Capture

> **Cross-cutting protocol** referenced by every skill that interacts with
> Daily Planner (tasks, outcomes, habits, goals, journal, finance, exercise,
> wellness, planning, learning, reporting, gamification).

## Purpose

Every time a skill touches Daily Planner — via the `DailyPlanner-*` MCP tools
or, less commonly, the backend APIs — we want to **continuously discover and
log friction**. The MCP surface is the canonical way Copilot interacts with
Daily Planner; the only way it gets better is if we treat every rough edge as
a captured backlog item, not a one-off workaround.

## When to capture a gap

Capture a backlog item the moment you notice any of these, even if you can
work around it for the current request:

| Symptom | Examples |
|---|---|
| **Missing MCP tool** | Had to fall back to `Invoke-RestMethod` against the backend API because no MCP tool exists for the operation. |
| **Missing backend endpoint** | The MCP tool exists but the backend doesn't expose the required field / action, so the tool can't be added. |
| **Missing field on response** | Tool returns a task / outcome / habit but omits a field you needed (e.g. `boardMetadata.estimatedEffort`, `dailyPlanner.lastSyncedAt`). |
| **Missing field on request** | Tool input doesn't accept a parameter you needed (e.g. cannot pass `dueDate` on create, no way to filter list by tag). |
| **Awkward shape / too many round-trips** | A single user intent required ≥ 3 MCP calls when one bulk call would suffice; or multi-step state changes that should be atomic. |
| **Bad default** | Tool silently caps lists at N, returns deleted items, or applies an unexpected filter. |
| **Unclear error** | `400 Bad Request` with no message about which field is invalid; `500` with no actionable detail; auth failure with stale token surfaced as a generic error. |
| **Inconsistent naming / shape** | One tool returns `taskId`, another returns `id`; one accepts `tags: string[]`, another `tags: string` comma-separated. |
| **Slow tool** | A read that should be ≤ 200ms takes > 2s under normal load. |
| **Doc gap** | Tool description omits a required side-effect, state-rule trigger, or precondition. |
| **Sync mismatch** | Disk-board snapshot and Daily Planner disagree in a way the existing skills can't reconcile. |

**Do NOT capture** as gaps:
- Transient network errors or expired tokens (re-auth and continue).
- User-data issues ("my task got the wrong tag" → fix the data, not the tool).
- Items already in the backlog with the same `tags: mcp-gap` and similar
  title. Search first with `DailyPlanner-list_tasks(tags="mcp-gap")` or the
  `review-backlog` filter; if it exists, optionally append a `Notes &
  Decisions` line via `append_task_note` instead of creating a duplicate.

## How to capture

1. **Call `DailyPlanner-create_task` directly** (do NOT invoke the
   `add-to-backlog` skill — capture must be inline and non-blocking).

   ```
   DailyPlanner-create_task(
     title       = "[MCP gap] <short imperative>",
     description = "<see template below>",
     priority    = "P3"  # P2 if it blocks a common workflow; P1 only if it blocks the user's current request
     type        = "Task",
     tags        = ["mcp-gap", "daily-planner", "<this-skill-name>"]
   )
   ```

2. **Description template** (markdown, copy verbatim and fill):

   ```markdown
   **Surfaced by:** <skill-name> (run on <YYYY-MM-DD>)
   **Repo context:** <cwd / repo / N-A>

   **What I tried:**
   <one or two sentences>

   **What was missing or broken:**
   <symptom — quote the exact tool name, field, error message if any>

   **Proposed fix:**
   - MCP: <new tool name + signature, OR new field on existing tool, OR fixed default>
   - Backend (if needed): <new endpoint, new field, schema change>
   - Docs: <what should the tool description say>

   **Impact:**
   <how often does this come up · which workflows it blocks · severity>

   **Workaround used (if any):**
   <so the next person hitting it has a path forward>
   ```

3. **Continue the user's original request.** Do not block on the gap unless
   it is a hard blocker for completing the user's intent. If you applied a
   workaround, mention it briefly in your reply so the user knows you didn't
   silently downgrade.

4. **Acknowledge the capture inline** in your reply with one line:

   ```
   📝 Captured MCP gap: [<integer-id>] <title>
   ```

5. **For high-impact gaps (P1/P2)**, also call
   `update_task_board_metadata(taskId, level="partial", estimatedEffort="unknown", proposedWorkflow="engineering-task", parallelizable=true, capturedBy="<this-skill-name>")`
   so `review-backlog` ranks it correctly.

## Surfacing captured gaps

The **`review-backlog`** skill auto-detects when it's run from inside the
`Sokokapu-Limited/daily-planner` repository (or any sibling microservice under
`C:\repositories\Sokokapu-Limited\`) and surfaces a dedicated **"🔧 MCP/API
Gaps"** section above the normal backlog table, listing all open items with
the `mcp-gap` tag across all boards. This gives the developer a fast loop:
work on Daily Planner → notice gap → capture → see it in the next
`review backlog` → fix it → MCP gets better.

Other ways to surface gaps:

- `get-tasks` skill: pass `tags="mcp-gap"` to see all captured gaps.
- Direct MCP: `DailyPlanner-list_tasks(tags="mcp-gap", status="!Completed")`.

## Critical rules

1. **Capture first, work around second.** The cost of a missed capture is
   compounding — every gap that goes unrecorded is one that keeps biting
   future sessions.
2. **One gap = one backlog item.** Don't bundle "missing field X" with
   "slow tool Y" into one item; they have different fixes.
3. **Always tag `mcp-gap` + `daily-planner` + `<skill-name>`.** All three.
   The first two enable filtering; the third tells us which skill's flow
   exposed the rough edge.
4. **Never let a gap-capture failure block the user's request.** If
   `DailyPlanner-create_task` itself fails, log the gap inline in the reply
   (so the user can capture it manually later) and proceed.
5. **The MCP-only rule still applies.** Even when capturing a "missing MCP
   tool" gap, you must use `DailyPlanner-create_task` (an MCP tool) to
   capture it — never `curl` / `Invoke-RestMethod`.
