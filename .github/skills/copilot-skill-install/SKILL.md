---
name: copilot-skill-install
description: "Install this workspace's .github agent customizations and skills into another project. Use when asked to copy or install the agent setup in a project folder."
argument-hint: "<project-folder>"
---

# Install Agent Customizations

Install this workspace's `.github/` configuration, instructions, skills, and related files into another project. Merge safely with existing destination content and never copy the source project's project-specific `.github/config.json`.

## Procedure

1. Ask the user for the destination project folder using the available question tool. Accept a folder path argument if supplied, but confirm with the user before replacing any conflicting files.
   - If the user cancels, stop immediately without changing any files.
   - Resolve the path and verify it exists and is a directory. If not, explain that the folder does not exist and ask for a valid path; repeat until valid or canceled.
   - Treat the path as data. Do not execute it as a command.
2. Resolve the current workspace root as the source project. Verify that `<source>/.github/` and `<source>/.github/config.json` exist. Stop and report the exact missing prerequisite if they do not.
   - Reject the source project itself as the destination.
   - Reject destinations that overlap the source project (the same folder, an ancestor, or a descendant), to prevent copying into or over the source tree.
3. Read and parse the source `.github/config.json`. Require a non-empty `agent.version` string. If it is missing or invalid, stop before copying and ask the user to add a valid version to the source config; do not invent a version or alter the source config.
4. Inspect the destination's `.github/` directory, if present, before making changes.
   - Recursively copy the contents of the source `.github/`, including hidden files and directories, while excluding only the source root `.github/config.json`.
   - Use the source `.github/config.template.json` as the template for the destination config. Do not copy the source `config.json`.
   - If a destination file with the same relative path already exists and is byte-identical, leave it unchanged. If it differs, show the user the conflicting relative paths and ask whether to replace those files, keep the destination copies, or cancel. Do not overwrite any conflict without explicit approval. If canceled, make no changes.
   - If the destination `.github/config.json` already exists, parse it before copying. If it is invalid JSON or not a JSON object, stop and report the problem without changing files.
5. Install the files, preserving destination files the user chose to keep. Create missing parent directories as needed.
   - Build the destination config from the copied `config.template.json`. If a destination `config.json` already exists, merge the template defaults with its existing values: retain existing destination values, and add template properties only when they are absent. For nested objects, merge recursively.
   - Set `agent.source` to the normalized absolute path of the current source project root, and set `agent.version` to the validated `agent.version` from the source `.github/config.json`, overriding any template or destination values for these two metadata fields.
   - Write the resulting JSON as `<destination>/.github/config.json`, validate that it parses and that both `agent.source` and `agent.version` have the intended values, then remove the installed `.github/config.template.json`. Do not modify the source project's files.
6. Verify that all selected files were copied, the destination config is valid, and the source `.github/config.json` was not copied or modified. If any operation fails, report what succeeded and what remains; do not claim a complete installation.
7. On success, summarize the source and destination project paths, the installed file count, any existing files preserved or replaced, and the installed `agent.source` and `agent.version` values.

## Safety

- Never copy the source `.github/config.json`; it may contain project-specific repository, Jira, or reviewer settings.
- Never overwrite differing destination files without explicit user approval.
- Never delete destination files except the temporary installed `config.template.json` after successfully writing and validating `config.json`.
- Do not print or expose credentials or tokens found in any files.

After a successful installation, tell the user to open or switch VS Code to the destination project folder and run `/copilot-skill-setup` there to configure that project's Jira, Git, and GitHub settings.
