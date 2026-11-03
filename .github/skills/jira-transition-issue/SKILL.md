---
name: jira-transition-issue
description: "Transition a Jira issue to a requested workflow status using Atlassian Rovo MCP. Use when asked to change a Jira ticket's status."
argument-hint: "<issue-key-or-number> <target-status>"
parameters:
  - name: argument1
    type: string
    description: "Jira issue key or ticket number (for example, TES-1 or 1)."
    required: true
  - name: argument2
    type: string
    description: "Exact destination status (quote it when it contains spaces, for example, \"CODE REVIEW\")."
    required: true
---

# Transition Jira Issue Status

Change a Jira issue's workflow status to the explicitly requested destination status. Use the authenticated Atlassian Rovo MCP tools; never use Jira REST directly or ask for an API token.

## Inputs

- Read the issue identifier from the first positional argument (`$argument1`), or the `issue`/`ticket` parameter.
- Read the destination status from the second positional argument (`$argument2`), or the `status`/`targetStatus` parameter.
- If an input is missing, ask only for that missing value before making any change. Treat a quoted multiword status, such as `"CODE REVIEW"`, as one argument.

## Procedure

1. Normalize the issue identifier:
   - Accept an issue key such as `TES-1` or a Jira browse URL and extract the key.
   - If given only a number, read `.github/config.json` and use its `project` property as the Jira project prefix (for this workspace, the configured prefix is `TES`). Construct the key as `<project>-<number>`.
   - If the prefix is missing/invalid or the input cannot be normalized, ask for a valid Jira issue key or URL. Never send a bare number to Jira.
2. Ensure the Atlassian MCP tools are available. If not, search for `jira-atlassian` tools before concluding the server is disconnected. If they remain unavailable, stop and tell the user to reconnect/reload the server in VS Code.
3. Resolve the Jira `cloudId`:
   - Use a known site URL or cloud ID from the active session if available.
   - Otherwise call `jira-atlassian-getAccessibleAtlassianResources` and select the user's accessible Jira site.
4. Call `jira-atlassian-getJiraIssue` for the normalized key, requesting `["summary","status","project"]`, with `responseContentFormat: "markdown"` and `updateHistory: false`. If this reports an error, do not attempt a transition.
5. Call `jira-atlassian-getTransitionsForJiraIssue` for that issue and cloud ID. Choose only a transition whose destination status name (`to.name`) matches the requested status, ignoring case and surrounding whitespace. Do not select by transition action name unless it is also the destination status.
6. If the current status already matches the requested destination, make no change and report that the issue is already in that status.
7. If no available transition reaches the requested status, make no change. Report the available destination status names and ask the user which valid status they want. Never guess a "close enough" status or use an unavailable transition.
8. Otherwise call `jira-atlassian-transitionJiraIssue` with the issue key, cloud ID, and the exact matched transition ID. Do not include unrelated fields or updates.
9. After the transition succeeds, call `jira-atlassian-getJiraIssue` again for `["summary","status","project"]` to verify the new status. Report success only when the returned status matches the requested destination. If verification fails or the transition tool reports an error, state that clearly; do not claim success.

## Safety and errors

- The user explicitly invokes this skill with an issue and destination status, which authorizes that single status change. Do not ask for redundant confirmation when both inputs are unambiguous and a matching available transition exists.
- Never modify other issue fields, add comments, or transition any issue other than the one identified by the user.
- Never request, print, or store API tokens. Authentication is through Atlassian Rovo MCP OAuth.
- On a 401/unauthorized error, stop and ask the user to reauthorize the Atlassian MCP server in VS Code.
- On missing access, invalid issue key, unavailable transition, or tool failure, explain the specific error and do not claim the issue was changed.

## Response

On success, report the issue key, summary, previous status, and verified new status. 

## Issue Moved

- **Issue**: [KEY] - [Summary]
- **Transition**: [old status] → [new status]
- **Jira Link**: <jira.baseUrl from config>/browse/[KEY]
```

## Error Handling

### Issue not found:
```
**Error**: Issue [KEY] not found. Verify the issue key is correct.
```

### API/MCP error:
```
**Error**: Failed to transition issue. Details: [error message]

Please verify:
1. The issue key is correct
2. You have permission to transition this issue
3. The Atlassian MCP connection is active
```

## Usage

```
/jira-transition-issue <JIRA_USER_STORY_ID> <JIRA_TARGET_COLUMN>
```

**Examples:**
- `/jira-transition-issue TES-123 IN PROGRESS`
- `/jira-transition-issue TES-456 DONE`
- `/jira-transition-issue TES-789 REVIEW`
