---
name: OpenSpec: Apply (Manual Workflow)
description: Implement an approved OpenSpec change and keep manual tasks in sync.
category: OpenSpec
tags: [openspec, apply, manual]
---
<!-- OPENSPEC:START -->
**Guardrails**
- Favor straightforward, minimal implementations first and add complexity only when it is requested or clearly required.
- Keep changes tightly scoped to the requested outcome.
- Ensure no files are modified outside the scope defined in the proposal and design documents.

**Steps**
Track these steps as TODOs and complete them one by one.
1. Read `changes/<id>/proposal.md`, `design.md` (if present), and `tasks.md` to confirm scope and acceptance criteria.
2. Work through the tasks in `tasks.md` sequentially, keeping edits minimal and focused on the requested change.
3. As you complete and verify each task, manually update `tasks.md` to check the box (`- [x]`). 
4. Confirm completion before moving to the archive stage—make sure every item in `tasks.md` is marked as done and reflects reality.
<!-- OPENSPEC:END -->