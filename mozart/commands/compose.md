---
name: compose
description: Compose a Mozart score from a goal. Brainstorms the design, generates validated score YAML, and offers to run it.
argument-hint: <goal>
allowed-tools:
  - Read
  - Glob
  - Grep
  - Write
  - Edit
  - Bash
  - Task
  - AskUserQuestion
---

# Mozart Compose

The user wants to compose a Mozart score. Their goal is provided as the argument.

**REQUIRED:** Use the `mozart:compose` skill to handle this request. The skill contains the full workflow — do not improvise.

If no goal was provided, ask the user what they want to accomplish.
