---
description: "Instructions for accessing Notion API and troubleshooting authentication. Use this skill when the Notion API returns 401 Unauthorized, when the user needs to regenerate their Notion token, or when any Notion MCP calls fail with authentication errors."
---

# Notion API Access Instructions

## Quick Fix: Token Expired (401 Unauthorized)

When Notion API calls return `401 Unauthorized`, the integration token has expired or been revoked. Follow these steps:

### Step 1: Generate a New Token
1. Open [notion.so/my-integrations](https://www.notion.so/my-integrations)
2. Click on the integration named **lifesuite**
3. Under "Internal Integration Secret", click **Show** then **Regenerate**
4. Copy the new token (starts with `ntn_`)

### Step 2: Update the MCP Config
The Notion MCP config lives at:
```
C:\Users\pingugi\.copilot\mcp-config.json
```

Update the `NOTION_API_KEY` value under `mcpServers.notion.env`:
```json
{
  "notion": {
    "env": {
      "NOTION_API_KEY": "<paste new token here>"
    }
  }
}
```

### Step 3: Restart the Notion MCP Server
The Notion MCP runs as `stdio` — it loads the token once at startup. After updating the config:
- In **VS Code**: Run `Developer: Reload Window` (Ctrl+Shift+P)
- In **GitHub Copilot CLI**: The server restarts automatically on the next session

### Step 4: Verify
Test the connection:
```
notion-API-get-self
```
Expected: `Bot name = lifesuite, Type = bot`

## How to Check the Current Token

The active token is always stored in the MCP config. To read it:
```powershell
(Get-Content "$env:USERPROFILE\.copilot\mcp-config.json" | ConvertFrom-Json).mcpServers.notion.env.NOTION_API_KEY
```

To quickly verify it works:
```powershell
$token = (Get-Content "$env:USERPROFILE\.copilot\mcp-config.json" | ConvertFrom-Json).mcpServers.notion.env.NOTION_API_KEY
$headers = @{ "Authorization" = "Bearer $token"; "Notion-Version" = "2022-06-28" }
(Invoke-RestMethod -Uri "https://api.notion.com/v1/users/me" -Headers $headers).name
# Expected output: "lifesuite"
```

> **Note for Copilot agent:** When the Notion MCP returns 401, read the token from the config above and test it directly via REST. If the REST call also fails, the token needs regenerating — guide the user through the steps below. If the REST call succeeds but the MCP still fails, the MCP server needs a restart.

## Important Notes

- **Token lifetime:** Notion internal integration tokens do not expire automatically, but they can be revoked or regenerated from the integrations page.
- **Page access:** The integration must be **connected to each page/database** it needs to access. In Notion, open the page → click `•••` → "Connections" → add **lifesuite**.
- **Integration name:** `lifesuite`
- **API Version:** `2022-06-28` (or later, as specified in Notion-Version header)

## Workspace Structure
Key pages the integration has access to:

| Page | Notion ID |
|------|-----------|
| Our Teams/Products | `16e978b5-b3d1-4251-818d-60528797580d` |
| Platform Health | `4e52f836-0ec4-43e7-8c09-129e26a6113c` |
| Team Documentation Standard | `31c891a6-db0d-8125-98b4-fab2e24f72ff` |
| My Role as Architect | `31c891a6-db0d-8155-a498-d6b892f9c713` |
