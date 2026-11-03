---
name: git-commit-task-changes
description: "Create a local Conventional Commit for the relevant changes from the current task. Use when asked to commit completed work."
argument-hint: "[optional paths to include]"
parameters:
  - name: argument1
    type: string
    description: "Optional file paths the user explicitly wants included in the commit."
    required: false
---

# Commit Task Changes

Create one atomic local Git commit for the changes relevant to the user's request. Follow Conventional Commits and never push.

## Commit message format

Use this subject format:

```text
<type>(<optional-scope>): <imperative description>
```

Allowed types:

- `feat`: add functionality
- `fix`: correct a bug
- `docs`: change documentation
- `chore`: maintenance or configuration
- `refactor`: restructure code without changing behavior
- `test`: add or correct tests

Subject requirements:

- Keep the complete first line under 50 characters.
- Use imperative mood (for example, `add user login`, not `added user login`).
- Be specific; never use vague subjects such as `updates`, `WIP`, or `fix stuff`.
- Do not end the subject with punctuation.
- Choose the type and concise description from the work performed, not merely the ticket title.
- Add a scope only when it improves clarity and still keeps the subject under 50 characters.

If explaining why the change was made is important, put that explanation in a concise commit body beneath the subject. Use the user's configured Git editor (`git commit` without `-m`) when an explanatory body is needed and the editor is available. Otherwise, ask before using a non-interactive multi-line message.

## Procedure

1. Confirm the current directory is inside the intended Git repository. If not, stop and ask the user to open the correct repository.
2. Identify which files and changes belong to the requested work:
   - First use the active conversation/task context and the files changed for that work. Do not include unrelated pre-existing or user changes.
   - If the task context does not identify the changes, inspect `git status`, `git diff`, `git diff --cached`, and untracked file paths/content. If a Jira issue key is present and the ticket context would help identify scope, use `jira-atlassian-getJiraIssue`; reuse matching Jira context already loaded instead of fetching it again.
   - Treat `$argument1` as an explicit request to include the named paths, but still inspect their diffs for secrets and unrelated changes.
   - If you cannot confidently distinguish requested work from unrelated changes, ask the user which paths/hunks to commit. Do not guess.
3. Inspect the complete diff for every proposed change, including staged, unstaged, and untracked content. Do not stage or commit secrets, credentials, API keys, passwords, local `.env` files, or unrelated work. If sensitive data is present, stop and explain.
4. Check the index before staging. If it contains staged changes outside the selected task scope, do not reset or unstage them; ask the user how to proceed. If a selected file mixes requested and unrelated edits, ask the user to separate the changes or specify the exact hunks. Never stage the entire repository with `git add .` or `git add -A`.
5. Always exclude `.github/`, `openspec/`, and `workarea/` from staging and commits, even if they contain task-related changes or are explicitly named. Check the repository's `.gitignore` and add ignore rules for these directories if any are missing. Stage only the explicitly selected, fully relevant paths outside those directories. Review `git diff --cached` and confirm every staged change belongs to this task and contains no sensitive data. If staging would include unrelated changes, stop and ask.
6. Determine the Conventional Commit type and an imperative subject from the actual diff and task context. Count the entire subject line, including type and scope, and keep it below 50 characters. If no clear message can be formed, ask the user rather than using a vague fallback.
7. Create the local commit only after the staged diff and message satisfy these rules. Do not add a co-author trailer or other trailer unless the user explicitly requests one.

8. Verify the commit with `git show --stat --oneline HEAD` and check `git status`. Report the commit hash, subject, and included paths. Clearly note any remaining changes that were intentionally left uncommitted.

## Safety rules

- The user's explicit invocation authorizes committing only the selected task changes. Do not commit unrelated staged or unstaged work.
- Never run `git push`, publish a branch, or push tags. All work remains local.
- Never amend, rewrite, or delete an existing commit unless the user explicitly requests it.
- Never use `git reset`, `git checkout`, or other destructive commands to clean up staging or discard work.
- Do not modify files merely to make them commit-ready without the user's request.
- If staging, message selection, or commit verification fails, report the error; do not claim success.

## Response

On success, provide the commit hash, Conventional Commit subject, included paths, and confirmation that nothing was pushed. If no commit was made, explain why and identify the next action needed.

Example:

```text
/git-commit-task-changes
```
