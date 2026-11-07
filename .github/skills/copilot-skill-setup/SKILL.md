---
name: copilot-skill-setup
description: "Set up Jira, Git, and GitHub CLI for a project by collecting non-secret settings, safely updating .github/config.json, and guiding authentication. Use when initializing or configuring this Copilot agent workspace."
---

# Set Up Copilot Agent Environment

Configure Jira project settings and GitHub defaults for the current project, set up the Jira MCP connection, and guide the user through Git and GitHub CLI installation and authentication.

## 0. Choose approval mode for this setup session

At the start of the workflow, before reading or changing project configuration, use the ask-user tool with one required single-choice field (`permission_mode`) and these options; set **Allow all** as the default:

- `allow_all` — label it **Allow all (recommended for fewer interruptions)**. Explain that it skips approval prompts for tool calls, including potentially destructive file edits, terminal commands, and external actions; recommend it only when the user trusts the project and understands the risk.
- `manual` — label it **Manual permissions**. Explain that it keeps approval prompts for actions that are not already approved.

Do not silently enable global auto-approval or change VS Code settings. This choice cannot be applied by the skill itself: explain that the user must select the matching mode in the current chat's **Permissions** picker. If the user chooses Manual permissions, continue without changing their current mode. If they choose Allow all, tell them where to select it and proceed only after they confirm; if they do not enable it, continue under their current approval behavior. Respect organization policy if a mode is unavailable.

## 1. Collect Jira settings and configure the API token

1. Read `.github/config.json` before editing. If it is missing, invalid JSON, or not a JSON object, stop and report the issue rather than overwriting it. Preserve all unrelated fields.
2. Ask the user for:
   - Jira base URL (for example, `https://example.atlassian.net`)
   - Jira project key (for example, `ABC`)
3. Validate that the URL is an absolute `https` URL with a host and that the project key contains only letters and numbers. Normalize the URL by removing trailing slashes and the project key to uppercase. If either is invalid, ask again before editing.
4. Do not ask the user to paste or send an API token in chat, and never read, display, log, or write the token itself to a file. Explain that Atlassian Rovo MCP uses OAuth and does not need this API token; the token is only for any separate integration that requires Jira API-token authentication.
5. Before asking whether `JIRA_API_TOKEN` is configured, explicitly show the user the following instructions. Do not replace these steps with only a yes/no environment-variable question:
   - Explain that this API token is separate from Jira MCP sign-in: the Atlassian Rovo MCP connection uses OAuth, while `JIRA_API_TOKEN` is only for a separate integration that requires Jira API-token authentication.
   - Open the Atlassian account security page: https://id.atlassian.com/manage-profile/security/api-tokens
   - Select **Create API token**, give it a recognizable label, create it, and copy it for immediate local entry.
   - In a local PowerShell window (not chat), run the following command and enter the token only at the secure prompt:

     ```powershell
     $secureToken = Read-Host "Enter Jira API token" -AsSecureString
     $tokenPointer = [Runtime.InteropServices.Marshal]::SecureStringToBSTR($secureToken)
     try {
       $plainToken = [Runtime.InteropServices.Marshal]::PtrToStringBSTR($tokenPointer)
       [Environment]::SetEnvironmentVariable("JIRA_API_TOKEN", $plainToken, "User")
     } finally {
       [Runtime.InteropServices.Marshal]::ZeroFreeBSTR($tokenPointer)
       Remove-Variable plainToken, secureToken -ErrorAction SilentlyContinue
     }
     ```

   - Explain that a persistent user environment variable is accessible to processes running as that user and should be treated as a secret. The secure prompt avoids putting the token in shell history, but Windows stores the environment value in a form available to that user. Do not print it to verify it.
   - After showing the explanation, link, and command, ask whether the user has completed the local command. If they already configured the variable, do not ask them to repeat the setup. If they cannot or do not want to set it, do not block Jira MCP OAuth setup; make clear the API-token-dependent integration will not be ready.
6. Update `.github/config.json` while preserving existing fields:
   - Set `jira.baseurl` to the validated Jira base URL.
   - Set `jira.projectKey` to the normalized project key.
   - Set `jira.apiToken` to the existing environment-variable reference format `{$env.JIRA_API_TOKEN}`. Never substitute the secret value.
   - Create the `jira` object if absent, or update only these Jira properties if present.
   - Validate the resulting JSON and ensure no token value was written.

## 2. Configure Jira MCP

Invoke the existing `jira-mcp-setup` skill and follow its procedure to configure or verify Atlassian Rovo MCP. Do not duplicate its setup instructions or add API tokens to `.vscode/mcp.json`. If that skill is unavailable, read and follow `.github/skills/jira-mcp-setup/SKILL.md`; if neither is available, stop this step and report the blocker.

Jira MCP server startup or OAuth authorization may take time or may not complete during this step. Do not block the rest of setup waiting for the server or Jira MCP tools: finish the Git/GitHub configuration and other setup steps, while clearly noting that Jira MCP is still starting or that authorization/live tools are not yet available. Tell the user that VS Code may show a **Sign in** / **Authorize** prompt later; if it does not, they can open the Command Palette > **Chat: Open Customizations** > **MCP Servers**, start or restart `jira-atlassian`, and complete authorization in the browser. At the end, report Jira MCP configuration status separately from live authentication/tool status. Do not claim the live connection succeeded unless a Jira MCP tool responds; invite the user to return to verify it once startup/authorization finishes.

## 3. Collect GitHub settings and update configuration

Before asking the user for GitHub defaults, inspect the current project and available CLI status yourself. Run `git --version` and `gh --version`; when `gh` is available, run `gh auth status --hostname github.com`. Do not ask the user whether Git or GitHub CLI is installed or authenticated before performing these checks.

Use the current project's `origin` remote and GitHub CLI to discover values:

- Run `git remote get-url origin` to identify the repository. Treat remote URLs as potentially sensitive: never repeat credentials or URL user-info if present.
- If GitHub CLI is installed and authenticated, run `gh repo view --json nameWithOwner,defaultBranchRef,url` from the project directory. Use `nameWithOwner` for the repository, `defaultBranchRef.name` for the base branch, and the repository URL/owner to derive the organization URL.
- If `gh repo view` cannot provide a value, try to determine the default branch from the remote's advertised HEAD (for example, `git ls-remote --symref origin HEAD`). Do not mistake the current checkout branch for the repository's default branch.
- Use existing non-placeholder values in `.github/config.json` as defaults. Prefer verified command results over guesses or template placeholders. Never invent a branch, repository, organization, or reviewer.
- Do not display raw command output that could contain credentials. Report only relevant non-secret values and whether checks succeeded.

Ask the user only for values that remain unknown after these checks:

- Base branch: describe it as “the branch new work should start from” and offer the detected default as the proposed answer.
- Repository: describe it as “the GitHub repository for this project” and offer the detected `owner/repo` as the proposed answer.
- Reviewer usernames, separated by commas. An empty response means no reviewers; preserve already-configured reviewers unless the user asks to change them.
- GitHub organization URL only if it cannot be derived from the verified repository URL. Explain that it is the GitHub page for the repository owner.

If GitHub CLI is installed but not authenticated, follow the sign-in instructions in section 4, then retry the repository lookup before asking the user to supply defaults. If a check fails or a value still cannot be found, say what was checked and ask only for that missing value.

Trim whitespace from branch and repository values, split reviewer names on commas and trim/remove empty entries, and validate the organization URL as an absolute `https` URL. Ask again only for invalid or unresolved values.

Update only the `github` properties `branch`, `repo`, `reviewers`, and `organizationUrl` in `.github/config.json`, preserving all unrelated config and existing fields. Store reviewers as a JSON array of strings. Normalize the organization URL to have exactly one trailing slash. Validate the final JSON and confirm the saved values without exposing secrets.

## 4. Set up Git and GitHub CLI

1. Use the `git --version` and `gh --version` results from section 3. If either tool is missing, report that result and explain the options below; do not ask the user to check something you can check yourself. Do not install software or change global Git settings without the user's approval.
2. If Git is missing on Windows, explicitly show the user the install options: download it from https://git-scm.com/download/win, or, if `winget` is available, run `winget install --id Git.Git -e` in a local terminal. Do not install it for them without approval. After installation, ask them to reopen the terminal and verify with `git --version`.
3. If GitHub CLI is missing on Windows, explicitly show the user the install options: download it from https://cli.github.com/, or, if `winget` is available, run `winget install --id GitHub.cli -e` in a local terminal. Do not install it for them without approval. After installation, ask them to reopen the terminal and verify with `gh --version`.
4. Check whether Git author name and email are configured without changing them. If identity setup is needed, explain that Git records these values in commit metadata and ask the user which name and email they want before showing commands. Explicitly show both choices:

   ```powershell
   git config --global user.name "Your Name"
   git config --global user.email "you@example.com"
   ```

   Explain that these settings apply globally to the user's repositories. If they want identity limited to this project, show the repository-local alternatives:

   ```powershell
   git config user.name "Your Name"
   git config user.email "you@example.com"
   ```

   Never invent or assume an identity. After the user runs their chosen commands, verify the configured identity without changing it.
5. If the `gh auth status --hostname github.com` check from section 3 indicates the user is not authenticated, explain that result and show them how to sign in: run `gh auth login --hostname github.com --web` in a local terminal, follow the prompts, complete authorization in the browser, then tell Copilot when finished so it can rerun `gh auth status` and retry repository discovery. Do not ask for or handle a GitHub token in chat. Do not claim authentication succeeded until the check passes.
6. Report missing tools, failed authentication, or access issues explicitly. Do not claim setup succeeded unless the corresponding checks pass.

## 5. Finish and summarize

Summarize:

- Jira base URL and project key
- Whether `JIRA_API_TOKEN` was configured locally, without displaying its value
- Jira MCP config and live OAuth connection status separately
- Git/GitHub CLI installation and authentication status
- Config file updated

Remind the user: **Add project-specific coding standards, compliance requirements, or security practices as Markdown files in `.github/guidelines/` if needed.**

## Safety and error handling

- Never put an API token, password, or OAuth secret in chat, `.github/config.json`, `.vscode/mcp.json`, or command-line arguments.
- Never overwrite malformed configuration or discard unrelated values.
- Do not claim a step succeeded when it was only instructed; verify tool availability, JSON validity, and live authentication separately.
- If a step is canceled or blocked, do not silently skip it; explain the remaining setup and continue only with independent steps the user has authorized.
