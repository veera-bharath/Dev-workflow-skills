---
name: OpenSpec: Proposal (Manual Workflow)
description: Scaffold a new OpenSpec change manually without the CLI.
category: OpenSpec
tags: [openspec, change, manual]
---
<!-- OPENSPEC:START -->
**Guardrails**
- Favor straightforward, minimal implementations first and add complexity only when it is requested or clearly required.
- Keep changes tightly scoped to the requested outcome.
- Identify any vague or ambiguous details and ask the necessary follow-up questions before editing files.
- Do not write any code during the proposal stage. Only create design documents (proposal.md, tasks.md, design.md, and spec deltas). Implementation happens in the apply stage after approval.

**Steps**
1. Review `openspec/project.md` and inspect related code or specs using standard search (e.g., `grep` or editor search) to ground the proposal in current behaviour.
2. Choose a unique verb-led `change-id` and manually create the directory `openspec/changes/<id>/`.
3. Create `proposal.md`, `tasks.md`, and `design.md` inside the new directory.
4. Map the change into concrete capabilities. Draft your new requirements and scenarios manually in `openspec/changes/<id>/specs/<capability>/spec.md` (create folders as needed).
5. Draft `tasks.md` as an ordered list of small, verifiable work items using standard markdown checkboxes (`- [ ]`).
6. Manually review your drafted specs against the master `openspec/specs/` folder to ensure no conflicting requirements exist before sharing the proposal.

**Reference**
- Search existing requirements manually with your code editor across `openspec/specs` before writing new ones.
<!-- OPENSPEC:END -->