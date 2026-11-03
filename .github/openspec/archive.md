---
name: OpenSpec: Archive (Manual Workflow)
description: Archive a deployed OpenSpec change and manually update root specs.
category: OpenSpec
tags: [openspec, archive, manual]
---
<!-- OPENSPEC:START -->
**Guardrails**
- Favor straightforward, minimal implementations first and add complexity only when it is requested or clearly required.
- Keep changes tightly scoped to the requested outcome.
- Do not archive the change until all code is deployed and the task list is 100% complete.

**Steps**
1. Identify the completed `<id>` folder inside `openspec/changes/`.
2. Open the `specs/<capability>/spec.md` files from your change folder. Manually copy the new or modified requirements and paste them into the corresponding master specification files located in the root `openspec/specs/` directory.
3. Ensure the root specs remain cleanly formatted and do not contain duplicate scenarios.
4. Move the entire `openspec/changes/<id>/` folder into the `openspec/changes/archive/` directory.
5. Do a final manual review of the master specs to ensure your merged changes accurately reflect the current state of the codebase.
<!-- OPENSPEC:END -->