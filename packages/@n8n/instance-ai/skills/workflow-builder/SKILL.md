---
name: workflow-builder
description: >-
  Load before calling build-workflow. Default path for all single-workflow
  work: new one-off workflows, existing-workflow edits, verification repairs,
  and workflow-local data tables. Write or edit a workspace source file, run
  workflow-sdk validate via workspace_execute_command, then call build-workflow
  with filePath. When the workflow creates or writes Data Tables, load
  data-table-manager first, then this skill. Do not load planning or
  create-tasks first. Load planning only when multiple coordinated workflows
  or shared cross-task data tables require a dependency-aware task graph.
recommended_tools:
  - read_file
  - write_file
  - edit_file
  - execute_command
  - build-workflow
  - workflows
  - nodes
  - data-tables
  - credentials
  - verify-built-workflow
  - executions
---

# Skill branch router

Use the branch reference that matches the task.

- Core concepts: [references/workflows/core.md](references/workflows/core.md)
- Implementation: [references/workflows/implementation.md](references/workflows/implementation.md)
- Verification: [references/workflows/verification.md](references/workflows/verification.md)
