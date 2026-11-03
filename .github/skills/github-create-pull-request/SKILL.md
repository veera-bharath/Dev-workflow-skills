---
name: github-create-pull-request
description: "Create a GitHub pull request from the current committed branch using GitHub CLI, configured repository, base branch, and reviewers. Use when asked to open or submit a pull request."
argument-hint: "[base-branch] [owner/repo] [optional PR title]"
parameters:
  - name: argument1
    type: string
    description: "Optional base branch override. Defaults to github.branch in .github/config.json."
    required: false
  - name: argument2
    type: string
    description: "Optional GitHub repository override (owner/repo). Defaults to github.repo in .github/config.json."
    required: false
  - name: argument3
    type: string
    description: "Optional explicit pull request title. Otherwise derive it from the task context and commits."
    required: false
---

# Create a GitHub Pull Request

Publish the current committed branch and create a pull request with GitHub CLI (`gh`). Creating a PR necessarily pushes the branch to GitHub; do not commit code as part of this skill.

## Inputs and configuration

- Read `.github/config.json`. Use `github.repo` and `github.branch` as defaults for repository and PR base branch.
- `$argument1` overrides the base branch, and `$argument2` overrides the repository. Use these only when supplied in this skill invocation; do not infer overrides from unrelated chat context.
- Use `$argument3` as the PR title if provided. Otherwise derive the title from the current task context and commits.
- Use `github.reviewers` as the requested reviewer list. If the property is missing, null, or empty, create the PR with the current user as assignee (`--assignee @me`) instead. Do not interpret an invalid configured reviewer as an empty list.
- If config is missing, malformed, or ambiguous, ask for the missing value before publishing anything.

## Procedure

1. Confirm the working directory is a Git repository and matches the configured GitHub repository. Resolve a name-only `github.repo` with `gh repo view` or exact repository search; continue only when the match is unique. Verify `gh auth status` and repository access. Do not switch to or clone an unrelated repository.
2. Resolve the current branch and ensure it is not the selected base branch. Resolve the correct Git remote for the configured repository; prefer `origin` only when it points to that repository.
3. Check `git status` and inspect staged, unstaged, and untracked changes. If there are any, stop and tell the user to commit intended work first (for example with the `git-commit-task-changes` skill). Do not stage or commit changes here.
4. Confirm the current branch has at least one committed change relative to the selected base branch. Fetch the base ref and compare commit ancestry / range. If there are no branch commits to propose, stop without pushing or creating a PR.
5. Determine PR title and body from the active task context and commit history:
   - Reuse already-loaded Jira ticket details for the matching issue, if available. Fetch Jira only if a relevant ticket key is known, the context is absent, and the details are needed to write an accurate PR summary.
   - Otherwise use the branch name and commit subjects/diffs. Do not invent behavior or claim tests that were not run.
   - Use a clear, concise title (Conventional Commit style when appropriate). Build the body with `Summary` and `Testing` sections; say when testing was not run or is unknown.
   - If the task scope or accurate PR description cannot be determined, ask the user before creating the PR.
6. Check whether a PR already exists for the current branch with `gh pr view`. If one exists, report its URL and current status; do not create a duplicate.
7. Push only the current feature branch and set its upstream, for example `git push -u <remote> <current-branch>`. This is required to publish a PR. Never push the base branch, tags, or unrelated refs. If push fails, stop and report the error; do not force-push.
8. Create the PR non-interactively with `gh pr create`, specifying `--base`, `--head`, `--title`, and `--body`, plus `--reviewer` for each configured reviewer. If the reviewer list is empty/missing, use `--assignee @me` instead. Quote and pass all values as data; do not execute config or user strings as shell code.
9. Verify the PR with `gh pr view` and report its URL, title, base/head branches, reviewers (or self-assignee), and current state. Only report success if the PR is confirmed.
10. Check CI state with `gh pr checks`. Report passing, pending, or failing checks accurately. Do not claim checks passed if they are pending or unavailable.
11. After reporting the PR and checks, offer the user these four choices, using `ask_user` when available:
    1. **Enable auto-merge and wait** — request auto-merge using an allowed repository merge method (prefer `--squash` unless repository policy indicates otherwise) and monitor checks/review status. Do not bypass required checks or reviews. GitHub does not allow an author to approve their own PR; do not attempt self-approval. If auto-merge cannot be enabled, explain why and wait only if the user still wants that.
    2. **Switch to the configured base branch** — switch to `github.branch` (or the base override used for this PR) only if the worktree is clean. If not clean, do not switch; explain that changes need to be saved first.
    3. **Wait** — wait for CI/review updates and report the latest observed state. Poll at a reasonable interval and stop if checks complete or the user asks to stop; do not merge unless option 1 was selected.
    4. **Tell me what to do** — provide concise next steps based on the observed checks/reviews, without changing branches or merging.
   Do not run any of these follow-up actions until the user selects one.

## Safety and errors

- Creating a PR authorizes pushing only the current feature branch to its matching remote. It does not authorize committing, force-pushing, pushing tags, merging, or bypassing branch protections.
- Never run `git commit`, `git push --force`, `gh pr merge` or an approval command unless the user selects the corresponding follow-up action. Option 1 authorizes enabling auto-merge and waiting, subject to repository policy; it does not authorize bypassing requirements or self-approval.
- Never publish local uncommitted changes. If local changes exist, stop before push.
- Do not overwrite or duplicate an existing PR.
- If push, PR creation, verification, or checks fail, report the exact failure and the resulting state (including whether the branch was pushed). Do not claim a PR exists unless verified.
- Do not expose credentials or authentication tokens.

## Response

Report the PR URL and metadata, whether the branch was pushed, and CI status. Then present the four follow-up choices and wait for the user's selection.

Example:

```text
/github-create-pull-request
```
