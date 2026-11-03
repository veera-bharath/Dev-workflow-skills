---
name: copilot-skill-setup
description: "Set up Jira, Git, and GitHub CLI for a project by collecting non-secret settings, safely updating .github/config.json, and guiding authentication. Use when initializing or configuring this Copilot agent workspace."
---

# Set Up Copilot Agent Environment

Configure Jira project settings and GitHub defaults for the current project, set up the Jira MCP connection, and guide the user through Git and GitHub CLI installation and authentication.

## 1. Collect Jira settings and configure the API token

1. Read `.github/config.json` before editing. If it is missing, invalid JSON, or not a JSON object, stop and report the issue rather than overwriting it. Preserve all unrelated fields.
2. Ask the user for:
   - Jira base URL (for example, `https://example.atlassian.net`)
   - Jira project key (for example, `ABC`)
3. Validate that the URL is an absolute `https` URL with a host and that the project key contains only letters and numbers. Normalize the URL by removing trailing slashes and the project key to uppercase. If either is invalid, ask again before editing.
4. Do not ask the user to paste or send an API token in chat, and never read, display, log, or write the token itself to a file. Explain that Atlassian Rovo MCP uses OAuth and does not need this API token; the token is only for any separate integration that requires Jira API-token authentication.
5. Guide the user to create an Atlassian API token:
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
   - Ask the user to confirm they completed the local command. If they cannot or do not want to set it, do not block Jira MCP OAuth setup; make clear the API-token-dependent integration will not be ready.
6. Update `.github/config.json` while preserving existing fields:
   - Set `jira.baseurl` to the validated Jira base URL.
   - Set `jira.projectKey` to the normalized project key.
   - Set `jira.apiToken` to the existing environment-variable reference format `{$env.JIRA_API_TOKEN}`. Never substitute the secret value.
   - Create the `jira` object if absent, or update only these Jira properties if present.
   - Validate the resulting JSON and ensure no token value was written.

## 2. Configure Jira MCP

Invoke the existing `jira-mcp-setup` skill and follow its procedure to configure or verify Atlassian Rovo MCP. Do not duplicate its setup instructions or add API tokens to `.vscode/mcp.json`. If that skill is unavailable, read and follow `.github/skills/jira-mcp-setup/SKILL.md`; if neither is available, stop this step and report the blocker.

Wait for the user to complete the Atlassian OAuth prompt and verify the live connection using the Jira MCP tools. Distinguish a valid configuration from a successful authenticated connection.

## 3. Collect GitHub settings and update configuration

Ask the user for:

- Base branch name
- Repository name (prefer `owner/repo` when known)
- Reviewer usernames, separated by commas; an empty response means no reviewers
- GitHub organization URL

Validate that the branch and repository are non-empty, trim whitespace, split reviewer names on commas and trim/remove empty entries, and validate the organization URL as an absolute `https` URL. Ask again for invalid values.

Update only the `github` properties `branch`, `repo`, `reviewers`, and `organizationUrl` in `.github/config.json`, preserving all unrelated config and existing fields. Store reviewers as a JSON array of strings. Normalize the organization URL to have exactly one trailing slash. Validate the final JSON and confirm the saved values without exposing secrets.

## 4. Set up Git and GitHub CLI

1. Check whether `git` and `gh` are available (`git --version`, `gh --version`). Do not install software or change global Git settings without the user's approval.
2. If Git is missing on Windows, guide the user to install it from https://git-scm.com/download/win or, if `winget` is available, run `winget install --id Git.Git -e` in their terminal.
3. If GitHub CLI is missing on Windows, guide the user to install it from https://cli.github.com/ or, if `winget` is available, run `winget install --id GitHub.cli -e` in their terminal.
4. Guide the user through Git identity setup only if needed. Ask for the name and email they want in commit metadata before suggesting:

   ```powershell
   git config --global user.name "Your Name"
   git config --global user.email "you@example.com"
   ```

   Explain that these are global settings; use repository-local `git config user.name` and `git config user.email` if they prefer settings limited to this project. Never invent or assume an identity.
5. Guide the user to authenticate GitHub CLI with `gh auth login --hostname github.com --web`, complete the browser flow, and verify with `gh auth status`. Do not ask for or handle a GitHub token in chat.
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
