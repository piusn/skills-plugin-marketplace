---
name: journal-trade
description: >
  Journal a closed (or interesting) trade to the trading-system position-learning store
  with optional screenshots. Use this skill when the user says "journal this trade",
  "journal a position", "log a trade learning", "add learning to position", "save trade notes",
  or similar. The skill calls the `journal_trade` MCP tool on `trading-mcp`, which persists
  to MongoDB (`position_learning`, Source = "manual-mcp") and uploads any screenshots to
  Azure Blob storage via the trading-api.
---

# Journal Trade

Capture an operator-authored learning note on a specific position, optionally with chart
screenshots. The result is a `position_learning` document with `Source = "manual-mcp"`
that survives alongside auto-generated agent learnings.

## When to Use

| User says | Action |
|---|---|
| "journal this trade" / "journal the XAU trade" | Run skill |
| "add a learning to position abc123" | Run skill |
| "log what I learned on the BTC trade" | Run skill |
| "save my notes on this position with these screenshots" | Run skill |

Do **NOT** use this skill for:
- Strategy refactors → use `engineering-task`
- Backlog capture of "I should investigate X" → use `add-to-backlog`
- Daily journaling of work activity → use `daily-journal`

## Prerequisites

- `trading-mcp` MCP server reachable at `http://100.127.42.20:3001` (production via Tailscale)
  or `http://localhost:3001` (local k8s after `deploy-local.ps1`).
- The MCP tool `journal_trade` is exposed by that server (visible in `tools/list`).
- The trading-api must have `TRADING_API_KEY` configured (it is, in prod). The MCP server
  forwards the same key as `X-Api-Key`; without it the API endpoint returns 401.

## Required Inputs

The `journal_trade` MCP tool requires:

| Field | Source | Notes |
|---|---|---|
| `positionId` | Mongo ObjectId (24-hex) | Ask if unknown. Can be looked up by symbol + open time. |
| `note` | Free-form learning text | The actual reflection / observation. |
| `symbol` | e.g. `XAUUSDz`, `BTCUSDz` | Should match the position's symbol. |
| `timeframe` | `M5`, `M15`, `M30`, `H1`, `H4`, `D1` | The TF the trade was executed on. |

Optional:

| Field | Notes |
|---|---|
| `outcome` | One of `win`, `loss`, `scratch`, `lesson`. |
| `strategy` | e.g. `SR-Reversal`, `Ema20Pullback`. |
| `screenshots` | **Preferred.** Array of `{ fileName, contentType, dataBase64 }`. |
| `screenshotPaths` | Filesystem paths — only works when MCP container has `MCP_SCREENSHOT_ROOT` set AND the path resolves under it. Production MCP does NOT mount a screenshot directory, so this path is unavailable in prod. |

## Workflow

### Step 0 — Verify MCP reachable

If you haven't seen `journal_trade` in the available tools list for this session, the
MCP server may be unreachable. Quick test:

```powershell
curl -sf -m 5 http://100.127.42.20:3001/health
# expect: {"status":"ok"}
```

If unreachable, surface the failure (server down, Tailscale disconnected, etc.) and stop.

### Step 1 — Gather inputs

Ask the user any missing required fields. Prefer ONE consolidated question that lists what
you need rather than 4 separate prompts.

**Resolving `positionId`:** if the user gives a symbol + rough time ("the XAUUSDz trade I
closed this morning"), look it up via the `local-mongo` MCP against the Hetzner
connection string:

```js
db.positions.find(
  { Symbol: "XAUUSDz", OpenTime: { $gte: ISODate("YYYY-MM-DDT00:00:00Z") } },
  { _id: 1, Symbol: 1, PositionType: 1, OpenPrice: 1, OpenTime: 1, ClosePrice: 1, Status: 1, RealizedProfit: 1 }
).sort({ OpenTime: -1 }).limit(5)
```

Present the candidates and let the user pick. `PositionType`: 0=BUY, 1=SELL. `Status`:
1=open, 2=closed.

### Step 2 — Handle screenshots

When the user attaches images in chat (system reminder will list them under
`copilot-image-*.png` paths in `C:\Users\Pius\AppData\Local\Temp\`):

1. Use the `view` tool to read each image — it returns base64 + MIME automatically.
2. Build a `ScreenshotPayload[]` for the `screenshots` parameter (NOT `screenshotPaths`,
   which the prod MCP can't resolve).
3. Each entry: `{ fileName: "<basename>", contentType: "<mime>", dataBase64: "<base64>" }`.

When the user gives file paths on their own machine:
- The MCP runs on Hetzner; it has no access to Windows paths. You still need to read each
  file locally (via `view` tool, which returns base64) and send via the `screenshots` field.

### Step 3 — Confirm with the user

Echo back the structured submission BEFORE calling the tool:

```
About to journal:
  Position : <positionId> (XAUUSDz BUY 1.23 @ 4538.50)
  Symbol   : XAUUSDz
  TF       : M5
  Strategy : SR-Reversal
  Outcome  : win
  Note     : <first 120 chars>...
  Screenshots: 2 attached (chart-entry.png 1.2 MB, chart-exit.png 980 KB)

Proceed?
```

Use `ask_user` with choices `["Yes, submit", "Edit the note first", "Cancel"]`.

### Step 4 — Call `journal_trade`

Once confirmed, call the MCP tool. On success it returns a JSON string containing the
created learning's id and screenshot URLs. Surface the response to the user:

```
✅ Journaled. Learning id: 681a5f...
   2 screenshots uploaded to Azure Blob.
```

### Step 5 — On failure

The tool returns a clear error string on validation failures (401, 400, file-too-large,
MIME-blocked, etc). Surface verbatim AND offer a remediation:

| Error | Remediation |
|---|---|
| 401 Unauthorized | `TRADING_API_KEY` mismatch between MCP and API. Check Hetzner `.env.production`. |
| 400 "screenshot too large" | Limit is 15MB per file / 60MB total. Ask user to compress / pick fewer. |
| 400 "unsupported content type" | Only `image/png`, `image/jpeg`, `image/webp`, `image/gif` allowed. |
| 404 "position not found" | The `positionId` doesn't exist. Re-query Mongo. |

## Example

User: "Journal the XAUUSDz SR-Reversal trade from this morning, screenshots attached.
Outcome was a small win. Note: 'SR level held cleanly on second touch; entry was a bit
late, exit was triggered by trailing rather than my target.'"

You:
1. Look up the position in Mongo (Symbol=XAUUSDz, OpenTime >= today).
2. Read attached screenshots via `view` tool → base64.
3. Confirm structured payload with `ask_user`.
4. Call `journal_trade` MCP tool.
5. Report success + learning id.

## Notes

- `Source = "manual-mcp"` distinguishes operator entries from agent-generated learnings.
  Both coexist on the same position document.
- Screenshots are uploaded with `PublicAccessType = None` — they're retrievable only via
  the authenticated screenshot proxy endpoint on trading-api.
- Multi-screenshot uploads are sequential in the MCP, not parallel. Expect ~1s/MB upload
  over Tailscale; warn the user if they attach >30MB.
- Manual learning limits: max 10 screenshots, 15 MB per file, 60 MB total.
