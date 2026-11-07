# Dev-workflow-skills

Reusable GitHub Copilot skills and agent customizations for a project. This guide walks through installing the tools, signing in to GitHub, and installing these skills into another project.

## What you need

- **Visual Studio Code** with **GitHub Copilot Chat** available and signed in. These skills are used from Copilot Chat; this repository does not install VS Code or a Copilot subscription.
- **Git**, to download the repository and work with project files.
- **GitHub CLI (`gh`)**, for skills that work with GitHub (such as creating or reviewing pull requests). You can install the skills without signing in to `gh`, but GitHub-related workflows will need authentication.

## 1. Install Git and GitHub CLI

### Windows

Open **PowerShell** from the Start menu and run:

```powershell
winget install --id Git.Git -e
winget install --id GitHub.cli -e
```

If Windows asks for permission, approve the installation. If `winget` is not available, download and install Git from [git-scm.com](https://git-scm.com/download/win) and GitHub CLI from [cli.github.com](https://cli.github.com/).

### macOS or Linux

Follow the installation instructions for your operating system on the official [Git website](https://git-scm.com/downloads) and [GitHub CLI website](https://cli.github.com/).

After installation, close and reopen PowerShell or your terminal, then check that both tools are available:

```text
git --version
gh --version
```

Each command should print a version number. If either command is not recognized, restart your computer and try again. If it still does not work, reinstall that tool using its official instructions.

## 2. Set your Git name and email

Git adds a name and email address to commits you create. Use the name and email you want associated with your work:

```powershell
git config --global user.name "Your Name"
git config --global user.email "you@example.com"
```

Replace the example values with your own. These settings apply to all projects on your computer. To set them only for one project, open that project's terminal and run the same commands **without** `--global`.

## 3. Sign in to GitHub

In PowerShell or a terminal, run:

```text
gh auth login --hostname github.com --web
```

When asked:

1. Select **GitHub.com**.
2. Select **HTTPS** as the preferred protocol for Git operations.
3. If asked whether Git should use your GitHub credentials, choose **Yes**.
4. Follow the prompt to open a browser, sign in to GitHub, and authorize GitHub CLI.

Check that sign-in worked:

```text
gh auth status
```

It should report that you are logged in to `github.com`. Never paste a password, access token, or browser code into Copilot Chat. Enter sign-in details only in the browser or terminal prompt that you opened yourself.

## 4. Download this repository

In PowerShell or a terminal, choose a folder where you keep projects, then download this repository. For example:

```powershell
cd $HOME\Documents
git clone https://github.com/veera-bharath/Dev-workflow-skills.git
```

In VS Code, select **File > Open Folder** and open the new `Dev-workflow-skills` folder inside Documents. If VS Code asks whether you trust the folder, confirm only if you downloaded it from the official repository above.

## 5. Prepare the local installer configuration

The installer needs a small local configuration file to identify the skills' version. It is not included in the repository because configuration can contain project-specific settings.

In the VS Code terminal, make sure the current folder is `Dev-workflow-skills`. If `.github/config.json` does not already exist, run:

```powershell
if (-not (Test-Path .github\config.json)) { Copy-Item .github\config.template.json .github\config.json }
```

This creates `.github/config.json` on your computer from the supplied template. If that file already exists, keep it and do not replace it. The installer uses its version information; it does not copy this source configuration file into your project. Do not add this local file to a commit.

## 6. Install the skills into your project

1. In VS Code, keep `Dev-workflow-skills` open as the current folder.
2. Open **GitHub Copilot Chat**.
3. Enter `/copilot-skill-install` in the chat box and send it.
4. When asked for the destination, enter the full path to the project where you want the skills installed. For example, on Windows: `C:\Users\YourName\source\my-project`.
5. Review any list of files that would be replaced. Choose whether to replace them, keep your existing files, or cancel. Existing files are not overwritten without your approval.

The installer copies the agent customizations into the destination project's `.github` folder and keeps that project's existing configuration. When it finishes, use **File > Open Folder** in VS Code to open the destination project.

## 7. Configure the installed skills (optional)

In Copilot Chat with your destination project open, run:

```text
/copilot-skill-setup
```

This setup guides you through project-specific GitHub and Jira settings and the Jira connection. You can skip it if you only want to use skills that do not need those settings. The setup may ask you to sign in to Jira separately; GitHub CLI sign-in does not sign you in to Jira.

## Troubleshooting

- **`git` or `gh` is not recognized:** finish installing the tool, close and reopen VS Code and its terminal, then try the version check again.
- **The installer says `.github/config.json` is missing:** return to step 5 and run the copy command from the `Dev-workflow-skills` folder.
- **The `/copilot-skill-install` command is not listed:** confirm GitHub Copilot Chat is available, `Dev-workflow-skills` is the folder open in VS Code, and the repository's `.github/skills` folder is present. Reopen the folder or restart VS Code and try again.
- **GitHub commands say you are not logged in:** run `gh auth login --hostname github.com --web` again and check with `gh auth status`.
- **You cannot access a project repository:** ask its owner to grant your GitHub account access. Signing in does not grant access by itself.
