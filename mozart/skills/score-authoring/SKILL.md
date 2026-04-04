---
name: score-authoring
description: Use when writing, reviewing, or fixing Mozart score YAML configs. Covers syntax (Jinja vs format strings), validation engineering, prompt design, fan-out architecture, and common pitfalls. Do NOT use for running/debugging jobs (use command instead).
---

# Mozart Score Authoring Skill

> **Purpose**: Write correct, effective Mozart score configs. Routes to the appropriate reference docs based on score complexity.

---

## Triggers

| Use This Skill | Skip This Skill |
|---|---|
| Writing new Mozart score YAML | Debugging existing Mozart errors (use `/mozart:command`) |
| Reviewing/fixing score configs | Running/monitoring jobs |
| Understanding available features | CLI operations only |
| Designing multi-stage workflows | |

---

## Quick Syntax Reference

**The #1 rule**: Jinja `{{ }}` in the **prompt pipeline** (templates, prelude/cadenza paths, capture_files). Python format `{}` in the **validation engine** (validation paths, commands, working_directory, skip_when_command).

| Field | Syntax |
|---|---|
| `prompt.template` | `{{ workspace }}` |
| `validations[].path` | `{workspace}` |
| `validations[].command` | `{workspace}` |
| `capture_files[]` | `{{ workspace }}` |

---

## Reference Docs

Load the docs matching your score's complexity tier:

| Tier | What to Load | Covers |
|---|---|---|
| **1** (simple pipeline) | `essentials.md` | Syntax, core variables, validations, config, YAML gotchas, pitfalls, pre-flight checklist |
| **2** (fan-out + synthesis) | `essentials.md` + `patterns.md` | + Design philosophy, fan-out patterns, Jinja mastery, prompt engineering, cross-sheet/parallel config |
| **3-5** (concert, self-chain, complex) | `essentials.md` + `patterns.md` + `advanced.md` | + Retry/rate limiting, circuit breaker, concert mode, post-success hooks, isolation, stale detection |

### Loading Instructions

Read the reference docs from `${CLAUDE_PLUGIN_ROOT}/docs/ref/`:

```
${CLAUDE_PLUGIN_ROOT}/docs/ref/essentials.md   — always load
${CLAUDE_PLUGIN_ROOT}/docs/ref/patterns.md     — load for tier 2+
${CLAUDE_PLUGIN_ROOT}/docs/ref/advanced.md     — load for tier 3+
```

**When this skill is invoked directly** (e.g., `/mozart:score-authoring`), load all three docs — the caller may need any level of detail.

---

## Additional Resources

- Example scores: `${CLAUDE_PLUGIN_ROOT}/docs/examples/` directory
- Fan-out gallery: [claude-compositions](https://github.com/Mzzkc/mozart-score-playspace)
- Operational guide: command skill (invoke via `/mozart:command`)
