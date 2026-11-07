---
name: jira-mcp-setup
description: "Configure or verify the official Atlassian Rovo remote MCP server in VS Code. Use when setting up Jira MCP, fixing the workspace MCP config, or checking its connection."
---

# Set Up Atlassian Rovo MCP

Configure VS Code to connect to Atlassian's official hosted Rovo MCP server. This is a remote HTTP server authenticated through Atlassian OAuth; it does not use a local `npx` process or Jira API token.

## 1. Interaction menu

Before making changes, present these numbered options:
1. **Configure or repair the workspace Jira MCP connection**
2. **Verify the existing connection without changing files**
3. **Cancel setup**

If the user chooses option 3, stop immediately and say: *"Setup canceled. No files were modified."*

## 2. Configure or repair (option 1)

1. Inspect `.vscode/mcp.json` in the workspace.
2. If it exists, parse it as JSON before editing. If it is invalid JSON, do not overwrite it; report the parse problem and ask the user to resolve it.
3. Preserve all unrelated top-level properties and server entries. Add or update only `servers["jira-atlassian"]` to this VS Code workspace configuration:

   ```json
   {
     "servers": {
       "jira-atlassian": {
         "type": "http",
         "url": "https://mcp.atlassian.com/v1/mcp"
       }
     }
   }
   ```

4. Do not put Jira site credentials, email, API tokens, or OAuth secrets in the workspace config. Do not create a local `.env` file.
5. Validate the resulting JSON and confirm the server entry has `type: "http"` and the exact Atlassian endpoint.
6. Give the user explicit steps to start the server and authorize it; do not assume they know how to trigger OAuth:
   - In VS Code, press `Ctrl+Shift+P` (`Cmd+Shift+P` on macOS), run **Chat: Open Customizations**, then select **MCP Servers**.
   - Find `jira-atlassian` in the workspace server list. Start it if it is stopped. If it is already running but not authenticated, stop and start it again.
   - When VS Code shows an Atlassian sign-in or authorization prompt, choose **Sign in** / **Authorize**, complete the flow in the browser, approve the requested access, and return to VS Code.
   - If no prompt appears, have the user return to this chat and report that. If the Jira MCP tools are available, invoke one to trigger the connection; if they are not, ask the user to restart the server from **MCP Servers**. The user should not need to edit the MCP config or provide credentials in chat.
   - Ask the user to report when they have completed authorization, or tell you if they are blocked. Do not continue as if OAuth succeeded while waiting.
7. After the user confirms authorization, verify by searching for the `jira-atlassian` tools or calling `getAccessibleAtlassianResources`. Report success only when the MCP tool responds; otherwise explain how to retry the sign-in from **Chat: Open Customizations** > **MCP Servers** and state what remains to be done. Do not claim authentication succeeded just because the JSON is valid.

## 3. Verify only (option 2)

1. Read `.vscode/mcp.json` without modifying it.
2. Confirm that the VS Code config uses top-level `servers`, and that `jira-atlassian` is an HTTP server at `https://mcp.atlassian.com/v1/mcp`.
3. Check connectivity using the available Atlassian MCP tools (for example, `getAccessibleAtlassianResources`). If the tool is not in the tool list, search for it before concluding it is unavailable.
4. Report config validity and live connection/auth status separately. If disconnected or unauthorized, give the user these steps: open the Command Palette (`Ctrl+Shift+P`, or `Cmd+Shift+P` on macOS), run **Chat: Open Customizations**, select **MCP Servers**, then start or restart `jira-atlassian`. Complete **Sign in** / **Authorize** in the browser if prompted, return to VS Code, and retry verification.

## 4. Important distinctions and security

- `.vscode/mcp.json` is the VS Code format: it uses the top-level `servers` object and an HTTP entry with `type` and `url`.
- Do not use the portable `.mcp.json` / Copilot `mcp-config.json` `mcpServers` schema in `.vscode/mcp.json`.
- Use Atlassian's hosted Rovo endpoint, not the community `mcp-atlassian` npm/Python package. Do not configure `npx`, `uvx`, or a local Jira REST API fallback in this skill.
- Do not ask the user to paste a token into chat. Do not inspect or print local API-token values. Atlassian Rovo MCP uses its OAuth authorization flow.
- Preserve unrelated existing configuration and never report a successful connection based only on config-file validation.

## References

- [Atlassian Rovo MCP server](https://www.atlassian.com/platform/rovo-mcp)
- [VS Code MCP configuration reference](https://code.visualstudio.com/docs/agents/reference/mcp-configuration)
- [VS Code: Add MCP servers](https://code.visualstudio.com/docs/agent-customization/mcp-servers)
