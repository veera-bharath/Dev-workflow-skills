---
name: copilot-workflow-managed
description: "Run the workspace's managed Jira-to-GitHub feature workflow: load and clarify a Jira ticket, draft and approve an OpenSpec proposal, implement and test, commit, create and review a PR, and track Jira status."
argument-hint: "<jira-issue-key-or-url>"
parameters:
  - name: argument1
    type: string
    description: "Jira issue key or browse URL for the feature to implement, for example TES-123."
    required: true
---

# Managed Jira-to-GitHub Workflow

Orchestrate the workspace's existing Jira, local branch, OpenSpec, commit, and GitHub PR workflows for one Jira issue. Follow the individual skills and workspace procedures linked below; do not invent substitute commands or skip their safety requirements.

## Inputs and configuration

- Accept a Jira issue key or browse URL from `$argument1`. If only a number is supplied, use `.github/config.json`'s `project` prefix to construct the key. If the config prefix is missing or the key is ambiguous, ask before continuing.
- Read `.github/config.json` and use its configured GitHub repository and base branch. Do not infer overrides from unrelated chat context.
- Reuse already-loaded Jira context for this exact issue; do not fetch it again if the key and relevant details are already in the active conversation.
- If a step has already been completed in this workflow and its result is verified in the conversation, do not repeat actions that would create duplicate comments, branches, commits, or PRs. Verify current state first.

## Workflow

### 1. Load and validate the Jira issue

Follow [jira-load-issue](../jira-load-issue/SKILL.md) and fetch the issue through `jira-atlassian-getJiraIssue` when matching issue context is not already loaded. Confirm the issue key and read its requirements, acceptance criteria, current status, and relevant metadata before making changes.

If the issue cannot be loaded, stop and report the reason. Do not update status or continue on an unverified issue.

### 2. Move the issue to In Progress

Follow [jira-transition-issue](../jira-transition-issue/SKILL.md) to move the issue to **In Progress**. The status skill must inspect current status and available transitions, and verify the resulting status. If the transition is unavailable or fails, stop and ask how to proceed.

### 3. Clarify requirements with the user

Present a concise summary of the Jira request and its acceptance criteria, then use `ask_user` to request corrections, clarifications, constraints, or additional context before planning implementation.

Before asking, tell the user that their response may be recorded as an issue comment and that they can opt out.
After the user responds:
- Incorporate the feedback into the working requirements.
- Post that feedback as a Jira comment using `jira-atlassian-addCommentToJiraIssue` unless the user opted out. Use the connected Jira `cloudId`, the issue key, and a concise comment headed `Implementation Notes`.
- If the user asks not to post it, do not comment; keep it in the workflow context.
- If the comment tool fails, report the failure and continue only if the user agrees.

### 4. Create a local feature branch

Follow [git-create-feature-branch](../git-create-feature-branch/SKILL.md), using the ticket key and loaded summary. The branch skill verifies the configured repository and base branch, requires a clean worktree, fetches and fast-forward pulls the base branch, then creates the feature branch locally.

Branch creation must not commit or push. If the repository, branch, or worktree checks fail, stop without discarding or stashing user changes.

### 5. Draft an OpenSpec proposal

Follow the workspace's manual [OpenSpec proposal procedure](../../openspec/proposal.md):
- Read all applicable compliance, coding-standard, and security-practice documents under `.github/guidelines/`, along with relevant project instructions/specs and existing behavior.
- Use those guidelines to shape the proposal, design, tasks, and any spec deltas. Identify relevant requirements and how the proposal addresses them.
- Create a unique change ID and draft proposal, design, ordered tasks, and any relevant spec deltas in the repository's established OpenSpec changes directory.
- Keep the proposal stage to design/specification only; do not implement code yet.
- Present a concise proposal summary and request explicit user approval.

If `.github/guidelines/` is missing or contains no applicable documents, note that project-specific guidelines were unavailable; do not claim compliance with rules that were not provided. If the proposal procedure or OpenSpec layout cannot be found, stop and ask rather than guessing file locations.

### 6. Get proposal approval

Before asking, tell the user that their response will be recorded as a Jira issue comment.
Use `ask_user` to offer **Approve and implement** or **Revise proposal**, with an optional field for feedback/clarifications.
- After the user approves, add a comment to the Jira issue using `jira-atlassian-addCommentToJiraIssue`:
  - **cloudId**: use the connected Jira `cloudId`
  - **issueIdOrKey**: the normalized Jira key from `$argument1`
  - **body**: format as:

    ```markdown
    **Implementation Notes (Copilot Workflow)**

    [user's feedback/clarifications]
    ```

- Include only feedback/clarifications the user actually provided; do not invent any. If they only approve and provide no feedback, use a concise body such as `Approved the proposal; no additional feedback provided.`, only comment once if the user approves and do not post again and again on each revise.
- On revision, collect the requested changes, update the proposal documents, summarize the revision, and ask again. Record each new substantive clarification once, avoiding duplicate Jira comments.
- If Jira comment posting fails, report the failure and ask how to proceed; do not claim the feedback was recorded.
- Do not implement until the user explicitly approves.
- If the user declines or pauses, leave the proposal unimplemented and report its change ID/path.

### 7. Apply the approved proposal

Follow the workspace's manual [OpenSpec apply procedure](../../openspec/apply.md). Before implementation, read all applicable documents under `.github/guidelines/` again in case they changed since proposal approval. Apply those compliance, coding-standard, and security requirements while implementing the approved proposal.

Read the approved proposal/design/tasks, implement tasks sequentially and within scope, and check off tasks only after each item is completed and verified. Validate the implementation against the applicable guidelines and report any requirement that cannot be met. Do not change files outside the approved proposal scope unless the user approves an update to scope.

If `.github/guidelines/` is missing or contains no applicable documents, explicitly note that project-specific guidelines were unavailable and rely on the repository's established conventions and general secure coding practices without claiming project-specific compliance.

### 8. Offer unit tests

Use `ask_user` to ask whether the user wants additional unit tests for the implementation.
- If **yes**, add tests using the repository's established conventions and run the smallest relevant test set.
- If **no**, skip adding tests and record that choice.
- Run relevant existing validation either way when feasible. Report commands and actual results; never claim tests passed if they were not run or failed.

### 9. Manual test and commit the task changes

Before committing, use `ask_user` to ask whether the user wants to test the implementation manually.

- If **No**, skip manual testing and continue to the commit instructions below.
- If **Yes**, pause and wait for the user to test. Offer these choices:
  1. **Tested OK** — continue to the commit instructions below.
  2. **Tell me what to fix** — ask for the observed problem and expected behavior, then make only the requested fixes, run relevant validation, and ask the user to manually test again.
  3. **Tell me what to do** — provide concise manual test guidance based on the feature and wait for the user's response; do not commit until they choose **Tested OK** or **No**.
- Do not infer a successful manual test. Continue to committing only after the user says it tested OK or explicitly chooses No.

Then follow [git-commit-task-changes](../git-commit-task-changes/SKILL.md). Use task context to identify only files and changes from this implementation; inspect staged, unstaged, and untracked changes for scope and secrets. Commit only the approved task changes using the skill's Conventional Commit rules. Do not add a co-author trailer unless the user asks. Never push in this step.

If unrelated changes are staged or the task changes cannot be isolated safely, stop and ask the user.

### 10. Create the pull request

Follow [github-create-pull-request](../github-create-pull-request/SKILL.md), using the repository/base branch from config. This step publishes the committed feature branch to GitHub as required to open the PR; it does not authorize creating commits, force-pushing, or pushing unrelated refs. Use the configured reviewers, verify the PR exists, and report CI status accurately.

After the PR is verified, post a Jira comment using `jira-atlassian-addCommentToJiraIssue` for this issue. Build the comment from the actual PR title/summary and PR number, using this format:

```markdown
**Pull request created (Copilot Workflow)**

<summary-of-pr>

PR: <github.organizationUrl>/<github.repo>/pull/<pr-number>

CC: @<reviewer>
```

- Read `github.organizationUrl`, `github.repo`, and `github.reviewers` from `.github/config.json`; use the actual PR URL/number returned by GitHub as the source of truth. Trim a trailing slash from `organizationUrl` before building the path to avoid duplicate slashes.
- Replace `<summary-of-pr>` with a concise, factual description from the PR title/body; do not invent scope or test results.
- If `github.reviewers` is missing, null, or empty, omit the entire `CC:` line.
- For each configured reviewer, look up their Jira identity using `jira-atlassian-lookupJiraAccountId`. If a Jira user ID/account ID is found, mention them using the Jira comment tool's supported mention format. If the ID cannot be resolved, include the reviewer as plain text (`@<reviewer>`). Do not fail the PR workflow solely because a reviewer cannot be matched.
- Do not add a Jira assignment or change the ticket's assignee; `CC` here means mentioning reviewers in the comment.
- Use the connected Jira `cloudId` and issue key. If comment posting fails, report the error and ask how to proceed; do not claim the comment was posted.

Do not continue if a PR cannot be created or verified. If an existing PR is found, inspect and reuse it rather than creating a duplicate; post the comment only if the corresponding PR comment has not already been added, to avoid duplicates. Defer the PR-creation skill's post-creation follow-up menu until this managed workflow reaches Step 13, so the review and Jira status steps happen before branch-switch/wait decisions.

### 11. Review the PR

Follow [github-review-pull-request](../github-review-pull-request/SKILL.md) and the applicable rules in that skill's `.github/guidelines/` folder. This review is read-only: report Critical, High, Medium, and Low counts and concise, line-referenced findings. Do not automatically post review comments, approve, or modify code.

If findings are reported, pause and ask whether the user wants them fixed. If approved, make only the selected fixes, test them, commit them with `git-commit-task-changes`, push only the current feature branch to update the PR, and rerun the review. Do not proceed to auto-merge while actionable findings remain unless the user explicitly accepts them.

### 12. Move the Jira issue to In Review

After a PR is verified and its review is complete, follow [jira-transition-issue](../jira-transition-issue/SKILL.md) to move the issue to **In Review**. The status skill must confirm the transition is available and verify the result. If not available, report the current Jira status and ask how to proceed; do not substitute a guessed status.

### 13. Ask what to do with the PR

After presenting the PR URL, review findings, and CI state, use `ask_user` to offer:

1. **Enable auto-merge and wait** — request GitHub auto-merge with an allowed method, and wait for required checks/reviews. Never attempt self-approval, bypass a protection rule, or merge directly. If auto-merge cannot be enabled, report why.
3. **Tell me what to do** — give next steps based on the actual checks and reviews without changing the branch or merge state.

Do not perform any follow-up action until the user chooses one. If auto-merge is selected and the PR later merges, verify the merged state before proceeding.

### 14. Ask whether to move the Jira issue to Done

Before asking, check the pull request's current state using the available GitHub tools. Report the verified state to the user:

- If the PR is merged, say that it is merged and ask whether they want to move the Jira issue to **Done**.
- If the PR is still open, pending review/checks, or otherwise not merged, say that it is still pending and ask whether they nevertheless want to move the issue to **Done**.
- If the PR state cannot be verified, explain that and ask whether they still want to request the transition.

Use `ask_user` to offer **Move issue to Done** or **Skip**. If the user chooses **Skip**, leave the Jira status unchanged and continue to Step 15. If they choose to move it, follow [jira-transition-issue](../jira-transition-issue/SKILL.md) with the exact target status **Done**; verify the transition and report the resulting status. If the transition is unavailable or fails, report the reason and continue to Step 15 without claiming it was changed.

### 15. Archive the OpenSpec change

Archive only after the PR is confirmed merged, the implementation is deployed/completed, and every task is checked off. Follow [OpenSpec archive procedure](../../openspec/archive.md), merge spec deltas into the master specifications without duplicates, and move the change folder to the archive location. Do not archive an open/unmerged PR, an unverified deployment, or an incomplete task list. If these conditions are not met, report why the change was not archived and continue to Step 16.

### 16. Switch to the configured base branch

After the Step 15 archive attempt, use `ask_user` to offer switching to the configured base branch:

1. If **Yes** — switch only if the local worktree is clean, then fetch and pull new changes; otherwise leave the branch unchanged and explain why.
2. If **No** — leave the branch unchanged.

## Stop conditions and recovery

- If any step fails, state which steps completed, which did not, and the precise error; pause for user direction rather than silently skipping or retrying non-idempotent actions.
- Do not undo successful prior steps to recover from a later failure. Resume by verifying current Jira, Git, OpenSpec, and PR state first.
- Never ask the user to paste API tokens or expose credentials.
- Never commit unrelated changes, force-push, self-approve, bypass checks, or merge without the user's explicit selection.
- If the user's request changes scope, update and re-approve the OpenSpec proposal before implementing out-of-scope work.

## Completion summary

When the workflow finishes, summarize the verified outcomes in a table:

| Stage | Result | Details |
|---|---|---|
| Jira issue | | Key, title, final status |
| Requirements | | Clarifications recorded or not posted |
| Branch | | Local branch and base |
| OpenSpec | | Change ID and archive status |
| Implementation/tests | | Scope and validation results |
| Commit | | Hash and Conventional Commit subject |
| Pull request | | URL, reviewers, CI/review state |
| Merge | | Merged, auto-merge pending, or not requested |
| Jira status | | Verified final status or unchanged |
| Switch to base branch | | Switched or not switched |

Mark incomplete stages as pending or blocked; never imply the entire workflow completed unless each required stage was verified.

## Example

```text
/copilot-workflow-managed TES-123
```
