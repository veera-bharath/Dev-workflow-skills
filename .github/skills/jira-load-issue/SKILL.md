---
name: jira-load-issue
description: Load a Jira ticket into the working context using the authenticated Atlassian Rovo MCP tools. Use for Jira issue keys, ticket URLs, or requests to load ticket details.
parameters:
  - name: ticket
    type: string
    description: "Jira issue key or URL (for example, TES-1 or https://your-company.atlassian.net/browse/TES-1)"
    required: false
  - name: argument
    type: string
    description: "Ticket identifier to load into context; typically the Jira issue key such as TES-1. If a plain number is provided, ask for the project key if needed."
    required: false
---

# Load Jira Issue Into Context

Use this skill when the user wants a Jira ticket loaded into the active working context so the agent can reason about the task without leaving the conversation.

## 1. Required behavior
When this skill is executed:
1. Determine the Jira issue key from the user input, `$argument`, or the `ticket` parameter; if none are provided, ask for it.
2. Accept a valid Jira issue key such as `TES-1`, a full Jira URL, or a ticket identifier that can be normalized into a Jira issue key.
3. If `$argument` is supplied, treat it as the ticket identifier to load and normalize it before querying.
4. If the user provides only a numeric value like `12345` and no project key, stop and ask for the Jira project prefix. **Do not attempt to call the Jira tool with a bare number.**
5. **Fetch live data through the Atlassian Rovo MCP server.**
   - If the Jira MCP tool is not already available in the tool list, search for `jira-atlassian getJiraIssue` before trying another method.
   - Call `jira-atlassian-getJiraIssue` with `issueIdOrKey` set to the normalized issue key, `fields` set to `["summary","description","status","assignee","reporter","priority","issuetype","project","labels","duedate"]`, `responseContentFormat` set to `markdown`, and `updateHistory` set to `false`.
   - Set `cloudId` to the Jira site URL (for example, `https://your-company.atlassian.net`) or its Atlassian cloud ID. If the site/cloud ID is unknown, call `getAccessibleAtlassianResources`, select the matching Jira site, and use its URL or ID.
   - Do not substitute a direct REST request, shell command, or local API-token credential for the Jira MCP tool.
6. Load the retrieved issue metadata into the conversation context and summarize the most important information.
7. Keep the result concise, structured, and immediately actionable for the user.

## 2. Security and auth rules
- Never ask the user to paste a Jira API token into chat.
- Never print, echo, or store the token in the workspace.
- Authentication is handled by the Atlassian Rovo MCP server's OAuth flow; do not request or use a Jira API token.
- If the Jira tools are unavailable, explain that the Atlassian MCP server is not connected and ask the user to check/reload it in VS Code. If a tool reports 401/unauthorized, ask the user to reauthorize the server.

## 3. Input normalization
- If the user provides a Jira URL, extract the issue key from the path before calling the MCP tool.
- If `$argument` is present, use it as the default ticket identifier.
- Treat a Jira ticket as normally represented by an issue key like `TES-1`, not a bare number.
- Normalize the value into the Jira issue key format when possible before requesting data.
- If the input is not a valid issue key or ticket identifier, ask: "Which Jira ticket should I load? Please share the issue key, ticket URL, or project prefix plus number."
- If multiple tickets are mentioned, focus on the first valid Jira issue key unless the user explicitly asks to load several.

## 4. Ticket context to load
Request the following fields from `jira-atlassian-getJiraIssue` and include those available in the response:
- Issue key
- Summary/title
- Description
- Status
- Assignee
- Reporter
- Priority
- Issue type
- Project
- Labels
- Due date or sprint metadata if available
- Acceptance criteria or custom fields only if they are clearly exposed and relevant

## 5. Response format
Return a compact result like this based on the data fetched:

```
Loaded Jira ticket: ABC-123
Title: <summary>
Status: <status>
Assignee: <assignee or Unassigned>
Priority: <priority>
Type: <issue type>
Project: <project>

Description:
<trimmed description>

Relevant context:
- <key point 1>
- <key point 2>
- <key point 3>
```

If the description is very long, trim it to the most relevant portion and list the rest as a follow-up suggestion.

## 6. Read-only behavior
This skill is intended to load ticket context only. It should not modify Jira data, create tickets, or change issue fields unless the user explicitly asks for a write operation and the appropriate tool is invoked.

## 7. Error handling
If the ticket cannot be found, the tool call fails, or the Jira connection is unavailable:
- State clearly that the ticket could not be loaded.
- Mention the likely cause based on the MCP tool's error output (e.g., invalid key, inaccessible issue, authorization failure, or disconnected MCP server).
- Offer one actionable next step: verify the key and access, reauthorize in VS Code, or reconnect the Atlassian MCP server as appropriate.
- If the tool is unavailable even after searching for it, stop and report that; do not claim a successful load or silently fall back to direct Jira API calls.

## 8. Example usage
- Load ticket `TES-1`
- Load Jira issue from https://mycompany.atlassian.net/browse/TES-1
- Load ticket using `$argument = TES-1`
- If the user says only `12345`, ask which Jira project prefix it belongs to before fetching the ticket