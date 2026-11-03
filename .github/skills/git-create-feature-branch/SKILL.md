---
name: git-create-feature-branch
description: "Create and check out a local feature branch from the latest configured base branch. Use when asked to create a ticket branch locally."
argument-hint: "<jira-issue-key> [short-ticket-description] [base-branch] [owner/repo]"
parameters:
  - name: argument1
    type: string
    description: "Jira issue key or URL, for example TES-1."
    required: true
  - name: argument2
    type: string
    description: "Optional short description for the branch slug. If omitted, use the Jira issue summary."
    required: false
  - name: argument3
    type: string
    description: "Optional base branch override. Defaults to github.branch in .github/config.json."
    required: false
  - name: argument4
    type: string
    description: "Optional GitHub repository override (owner/repo). Defaults to github.repo in .github/config.json."
    required: false
---

# Create and Check Out a Local Feature Branch

Create a **local-only** feature branch from the latest version of the configured base branch. Fetch and fast-forward pull the base branch, then create and check out the feature branch from it. Never commit or push.

## Inputs and configuration

- Read the Jira ticket key from `$argument1` or an explicit `issue`/`ticket` parameter. Accept a key like `TES-1` or a Jira browse URL and extract the key.
- Use `$argument2` or an explicit description/title parameter as the branch description when supplied. If omitted, reuse the summary already loaded in the active conversation for this same issue key; fetch Jira only when matching ticket context or its summary is not already available.
- Read `.github/config.json`. Its `github.repo` and `github.branch` values are the defaults for repository and base branch.
- Use `$argument3` or an explicit `baseBranch` parameter to override the base branch, and `$argument4` or an explicit `repo` parameter to override the repository. Otherwise use the config values. Do not infer overrides from unrelated chat context.
- If a required config value is missing, malformed, or ambiguous, ask for the missing value before creating anything.

## Branch naming

Create the branch as:

```text
feature/<lowercase-jira-issue-key>-<short-description-slug>
```

For example, `TES-1` with summary "My First Ticket" becomes `feature/tes-1-my-first-ticket`.

- Normalize the key and description to lowercase.
- Convert runs of spaces and punctuation to a single hyphen; remove leading/trailing hyphens.
- Keep the description to a few meaningful words; omit stop words only when needed to keep it concise.
- Keep the `feature/` prefix and ticket key. If the description is empty after normalization, ask the user for a short description rather than creating a key-only branch.
- Always follow this naming pattern; user overrides apply to the repository and base branch, not the generated branch name.

## Procedure

1. Validate inputs and read `.github/config.json`. Do not create a branch until the ticket key, repository, base branch, and resulting branch name are known.
2. Resolve a missing description:
   - First check whether the active conversation already contains Jira details for this exact issue key. Reuse its summary to derive the branch description; do not make a duplicate Jira request when the matching summary is already loaded.
   - Otherwise search for the Atlassian Jira MCP tool if it is not already available, then call `jira-atlassian-getJiraIssue` for the key, requesting `["summary"]`, `responseContentFormat: "markdown"`, and `updateHistory: false`.
   - If the tool is unavailable or the issue cannot be read, ask the user for a short description. Never invent ticket details.
3. Ensure the current directory is a Git working tree for the configured GitHub repository:
   - Resolve `github.repo` to `owner/name`. If config contains only a name, use `gh` to locate an exact accessible repository match and continue only if it is unique.
   - Use `git remote -v` to find a configured remote whose fetch URL matches that GitHub repository. Prefer `origin` if multiple remotes point to it; if no matching remote exists, stop and ask the user to open or clone the correct repository. Never run branch operations in an unrelated repository.
   - Verify repository access/readability (for example with `gh repo view <owner/name>`). If access or required tools are unavailable, stop and explain the blocker.
4. Check the worktree before switching or pulling. If there are staged, unstaged, or untracked changes, stop without modifying anything and ask the user to save or stash them. Never stash, discard, reset, or commit user changes.
5. Inspect the current local branch with `git branch --show-current`. If it is not the selected base branch, switch to the selected base branch. If that local base branch does not exist, fetch the matching remote base ref and create a local tracking branch from it. Stop if checkout would overwrite changes or if the base ref cannot be found.
6. Fetch the selected base branch from the verified repository remote (for example, `git fetch <remote> <base-branch>`), then fast-forward pull it (for example, `git pull --ff-only <remote> <base-branch>`). Never use a merge-commit or rebase pull. If fast-forward is not possible, stop and report that the base branch needs manual attention; do not reset it or attempt to resolve conflicts automatically.
7. Confirm the current local branch is the selected base branch and its worktree remains clean. Check whether the desired feature branch already exists locally. If it does, stop without switching to or changing it.
8. Create and check out the local branch from the updated base using `git switch -c <branch-name>`. Treat names as data, quote shell values safely, and do not execute user-provided text as shell code.
9. Verify `git branch --show-current` equals the new branch name and that the branch points to the updated base commit. Report success only after verification.

## Safety and errors

- The user's explicit invocation authorizes creating and checking out only the requested local feature branch, plus fetching and fast-forward updating the selected local base branch.
- **Never run `git push`, `gh api` to create a remote ref, or any other operation that publishes the branch. Never commit any code. All branch creation remains local.**
- Never force-update, delete, reset, merge, or rebase branches. Never alter any other branch beyond the requested fast-forward pull of the selected base.
- Do not print or expose credentials or authentication tokens.
- If repository/base ref lookup, fetch, pull, checkout, or branch creation fails, report the exact failure and do not claim success.

## Response

On success, report the GitHub `owner/repo`, local branch name, and source base branch. 

```
## Branch Created

- **Branch**: feature/<branch-name>
- **Based on**: <github.branch from config>
- **Issue**: [$argument1] - [Summary]
- **Remote URL**: <github.organizationUrl>/<github.repo>/tree/<branch-name-url-encoded>
```

Explicitly confirm that no commit or push was performed. For failures, state whether the base branch was updated and whether a feature branch was created, then provide the relevant next step.

## Usage Example

Example:

```text
/git-create-feature-branch TES-1 "my first ticket"
```
