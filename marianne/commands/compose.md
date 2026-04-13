---
name: compose
description: Compose a Marianne score from a goal. Brainstorms the design, generates validated score YAML, and offers to run it.
argument-hint: <goal>
disable-model-invocation: true
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

Invoke the marianne:composing skill and follow it exactly as presented to you
