# Coordination Protocol — Technique Module

## Purpose

The coordination protocol teaches agents how to use the shared workspace
for self-organization. Without explicit hierarchy, agents coordinate through
artifacts — plans, findings, decisions, and status documents that live in
shared directories. The coordination protocol makes this work smoothly.

## Shared Workspace Structure

```
workspace/
  shared/
    active/       <- Curated live context (token-budget managed)
    specs/        <- Relevant specifications copied here by agents
    plans/        <- Coordination plans, priorities, task claims
    findings/     <- Shared finding registry (mateship protocol)
    decisions/    <- Architectural decisions, trade-off records
    directives/   <- Composer notes, human overrides
    techniques/   <- Shared patterns, method documents
  agents/
    {name}/
      work/       <- Agent's private working directory
      reports/    <- Per-cycle reports
      cycle-state/<- Recon, plan, aar artifacts per cycle
  collective/
    memory.md     <- Shared memory (append-only)
    tasks.md      <- Task registry
    status.md     <- Project status
```

## The Active Folder

The `shared/active/` directory is the live cadenza — a curated space that
agents manage together. Its contents are loaded into every agent's context
during applicable phases.

### Token Budget

The active folder has a token budget (default: 8000 tokens). When contents
exceed this budget, a size signal fires and agents should curate:
- Archive stale artifacts to `shared/archive/`
- Compress verbose documents
- Remove duplicates
- Keep only what's relevant to current cycle priorities

### Curation Rules

1. **Put relevant artifacts here.** If other agents need to see it now,
   put it in active.
2. **Archive when done.** Completed work artifacts move out of active.
3. **Don't dump everything.** Active is curated, not a file dump.
4. **Size signal = action needed.** When the budget warning fires, curate.

## Claim-Before-Work Protocol

Before starting work that could overlap with other agents:

1. **Read shared/plans/** to see what others are working on.
2. **Write your plan** to `agents/{name}/cycle-state/{name}-plan.md`.
3. **If overlap detected**, check the other agent's plan for coordination
   opportunities rather than duplicating work.
4. **Update collective/tasks.md** with your claimed work items.

This prevents two agents from independently fixing the same bug or
implementing the same feature.

## Composer Directives

The human (or AI composer) writes to `shared/directives/`. All agents read
this during recon. Directives take priority over self-organized plans.

Common directive types:
- **Priority shift**: "Focus on security audit this cycle."
- **Pairing instruction**: "Agent-A and Agent-B should coordinate on the API refactor."
- **Scope change**: "Skip play phase until the P0 backlog is cleared."
- **Architecture decision**: "Use Protocol-based abstractions for the new module."

Agents must check directives before planning their cycle.

## Glob Listing Strategy

For token efficiency, agents get a lightweight glob listing of all shared
directories in their prelude (what exists, not what it contains). This map
costs ~2000 tokens and gives agents awareness of the full shared space.

The full content of `shared/active/` is loaded as cadenza. Other directories
are accessible via filesystem techniques when needed.

## Task Registry (collective/tasks.md)

The task registry is a shared document where agents track work items:

```markdown
## Active Tasks

- [ ] [agent-a] Refactor config loading — claimed cycle 12
- [ ] [agent-b] Implement validation pipeline — claimed cycle 12
- [x] [agent-c] Security audit of auth module — completed cycle 11

## Unowned

- [ ] Fix flaky test in test_daemon.py — P1, filed cycle 10
- [ ] Update API documentation — P2, filed cycle 9
```

Agents update this during plan and AAR phases.

## Guidelines for Agents

When this technique is active in your phase:

1. **Read the landscape first.** During recon, read shared/plans/, the task
   registry, and any directives. Understand the coordination state.
2. **Claim before working.** Write your plan. Update the task registry.
   Other agents will see it and avoid collision.
3. **Use active/ judiciously.** Put things there that other agents need now.
   Remove things that are done or stale.
4. **Respect directives.** Composer overrides take priority. If a directive
   changes your planned work, adapt.
5. **Communicate through artifacts.** Your recon report, plan, and AAR are
   how other agents know what you're doing. Write them well.
6. **Coordinate, don't compete.** Two agents finding the same bug is waste.
   Check shared/findings/ before investigating.
7. **Update shared status.** If you complete work that affects others, update
   collective/status.md and relevant plans.
