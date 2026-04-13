---
name: composing
description: Use when the user wants to compose a Mozart score from a goal, turn an idea into a multi-stage AI workflow, or design orchestration YAML. Triggered by /mozart:compose or when the user describes work that would benefit from multi-stage, multi-agent orchestration.
---

# Score Composition

Composition transforms a goal into a score — a program that orchestrates intelligences through coordinated work. The quality of your composition determines whether downstream agents succeed or fail. Every design decision affects work you will never see performed by agents you will never interact with.

This skill teaches the act of composition itself. The score-authoring skill covers YAML syntax and mechanics. The usage skill covers running and debugging. This skill covers the cognitive process: understanding what to build, recognizing the forces that shape the solution, and designing a score whose structure faithfully serves the work.

Scores are general-purpose. Software, research, creative work, business operations, education — any domain where work can be decomposed into stages with validation. The domain shapes the content; the composition process is the same.

---

## Principle Zero

The user is the authority. You propose; they decide. Present your reasoning, offer alternatives, get approval before generating YAML.

---

## Vocabulary

A **score** is a YAML configuration that declares what work to do, which instruments to use, how to validate outcomes, and how to recover from failures.

A **sheet** is one agent performing one task — the atomic unit of execution. A **stage** is a logical phase that may expand into multiple sheets via **fan-out** — parallel instances of the same work on different data using the same instrument. If instances need different instruments, they are separate stages, not fan-out. In YAML, stages are declared via `movements:` (with per-movement names and instrument assignments) and `sheet.total_items` (the pre-expansion stage count). The `stage` template variable gives each sheet its logical phase number.

An **instrument** is an AI backend or CLI tool with specific capabilities and cost profile. Run `mzt instruments list` to see what's available.

**Preludes** inject shared context into every sheet. **Cadenzas** inject per-sheet context. Both guarantee the agent has what it needs — telling an agent to "read a file" is unreliable.

A **concert** chains multiple scores via on_success hooks. The **workspace** is where all artifacts live — the score's shared memory. The **conductor** is the daemon that manages execution.

The musical vocabulary is load-bearing. Scores, sheets, movements, voices, instruments, preludes, cadenzas, concerts, conductors — these name architectural concepts, not decorations. When the naming feels wrong, the architecture may be wrong.

### Forces

Every coordination problem has a force profile. Forces drive pattern selection.

| Force | The question it asks |
|-------|---------------------|
| **Information Asymmetry** | Do different agents need to know different things? |
| **Finite Resources** | Does the work exceed what one agent can handle well? |
| **Partial Failure** | Can components fail independently? |
| **Exponential Defect Cost** | Do problems found late cost more than problems found early? |
| **Producer-Consumer Mismatch** | Do stages produce in formats the next stage can't directly consume? |
| **Instrument-Task Fit** | Do different tasks need fundamentally different capabilities? |
| **Convergence Imperative** | Does iterative work need domain-specific stopping criteria? |
| **Accumulated Signal** | Must evidence build to a threshold before triggering change? |
| **Structured Disagreement** | Are single perspectives unreliable for this problem? |
| **Progressive Commitment** | Is full commitment before validation risky? |

The forces reference at `scores/rosetta-corpus/forces.md` maps forces to generators and generators to patterns.

### The Rosetta Corpus

The corpus at `scores/rosetta-corpus/` contains 56 coordination patterns organized by scale — within-stage (prompt structure), score-level (sheet arrangement), concert-level (multi-score coordination), and cross-cutting categories (adaptation, instrument-strategy, iteration, communication).

**Before using a pattern, read its file.** Each file at `scores/rosetta-corpus/patterns/<name>.md` contains:

- **Problem** and **Signals** — when to reach for this pattern
- **Forces** and **Generators** — what structural properties produce it
- **Stages** — the shape: how many stages, what each does, what instrument capability serves each
- **Composes With** — other patterns this naturally combines with
- **Core Dynamic** — the mechanism, not just the name
- **Failure Mode** — how this pattern breaks
- **Marianne Score Structure** — a YAML snippet showing the shape

The INDEX at `scores/rosetta-corpus/INDEX.md` shows all patterns with their problems and signals. Pattern filenames are kebab-case versions of the display name (e.g., "Fan-out + Synthesis" → `fan-out-synthesis.md`, "Commissioning Cascade" → `commissioning-cascade.md`). The selection guide at `scores/rosetta-corpus/selection-guide.md` maps problem types to pattern compositions. The glossary at `scores/rosetta-corpus/glossary.md` defines all terms.

Patterns are structural moves, not templates. They describe coordination shapes that exist because coordination requires them. When you compose multiple patterns, you create a shape none of them have individually.

---

## The Workflow

```
UNDERSTAND → QUESTION → DESIGN GATE → COMPOSE → REVIEW → OFFER
```

### Phase 1: Understand

Study the user's goal:
- What did they say they want?
- What do they probably need but didn't articulate?
- What would make the output wrong?

**Offer to investigate the project** before designing. Investigation produces better scores — conventions discovered now prevent validation failures later. But the user may already have the context you need, or the goal may not require project-specific knowledge.

If the user wants investigation, two areas matter:

1. **Venue context** — project structure, spec corpus (`.marianne/spec/`), conventions from project docs, available instruments. Produces a venue profile that informs composition.

2. **Codebase analysis** (for code goals only) — relevant files, test infrastructure, build/lint/type-check commands, integration points, complexity assessment. Skip for non-code goals.

If your platform supports dispatching subagents, run both investigations in parallel for speed. The `venue-explorer` and `codebase-analyzer` agents in this plugin are designed for this. If subagents aren't available, do the investigation yourself — read the project docs, check the directory structure, find the test commands. The work is the same either way.

When investigation completes, extract conventions the score must respect, constraints that affect design, and (for code goals) specific files, test commands, and integration points.

### Phase 2: Question

Ask at least one question. Even when the goal seems clear, confirm your understanding. Questions that reveal misalignment save more time than they cost.

Focus on what the user hasn't considered — missing validation strategies, incomplete decomposition, unstated constraints. Cover their gaps.

Skip only if the user explicitly asks you to. Prefer multiple choice. One question per message.

### Phase 3: Design Gate

Present the score architecture before generating YAML. This is non-negotiable.

Show your reasoning: what forces you identified, which patterns address them, how the stages decompose, what instruments serve each stage, what workspace you'll use, and how validations prove outcomes.

For straightforward compositions, a concise summary suffices. For complex ones, show the full stage breakdown with reasoning.

Wait for approval before proceeding.

### Phase 4: Compose

Generate the score YAML. This phase applies the composition methodology — see the next section.

Before writing YAML, load the score-authoring reference. Invoke the `score-authoring` skill, or read the tiered reference docs directly:
- `essentials.md` — needed for all scores (syntax, validations, config, pitfalls)
- `patterns.md` — for fan-out, multi-stage design, Jinja, prompt engineering
- `advanced.md` — for concerts, self-chaining, isolation, stale detection

These live in this plugin's `docs/ref/` directory.

### Phase 5: Review

Have the score reviewed adversarially — dispatch `score-reviewer` if available, or review it yourself with an adversarial stance: assume the score has problems until proven otherwise. Check workspace safety, syntax correctness, validation quality, and first-run safety.

Fix all critical and important issues before presenting to the user.

### Phase 6: Offer

Present the score. Show where it will be saved, what it does when run, and any caveats from the review.

If the conductor is running, offer to submit it. If not, show the run command.

**Save location** — infer from context:
- Scores for ongoing development → `scores-internal/`
- Scores that are part of project operations → `scores/`
- Example scores for documentation → `examples/`
- If unclear, ask the user

---

## The Composition Methodology

This is the heart of Phase 4. It describes how to move from an approved design to a correct score.

Composition is judgment, not procedure. The steps below give structure to that judgment, but the order is not rigid — decomposition might reveal wrong pattern selection, force analysis might change your understanding of the goal. Move fluidly. The goal is a score that serves the user's intent, not a score that satisfies a checklist.

### Define the Work

Before touching patterns or YAML:

- What is the input? What data, files, or state does the work start from?
- What is the output? What must exist when the score completes?
- What are the units of work? Can they be processed independently?
- What makes output wrong? What does failure look like?
- What varies across units? Do some require different treatment?
- What must be shared? Do agents need common context or standards?
- What depends on what? Which work must complete before other work begins?

These answers determine everything that follows.

### Analyze Forces

For each of the 10 forces in the table above, ask whether it's active in your problem. Note what evidence makes each force active or inactive. This evidence grounds your pattern selection — without it, pattern choice is name recognition.

The selection guide at `scores/rosetta-corpus/selection-guide.md` organizes patterns by problem type and recommends compositions. Start there when the force profile matches a known problem type.

### Select Patterns

Active forces point to candidate patterns. For each candidate, **you must read the full pattern file** at `scores/rosetta-corpus/patterns/<name>.md`. Not a summary. Not from memory. The file itself. Pattern files contain the structural shape (stages, instrument guidance, DAG topology) that you need in the next step — without reading them, you cannot derive the score's structure and will produce a score that names the pattern without implementing it.

After reading each pattern file:

1. **Check the signals.** Do they match your situation? Does the core dynamic address the forces you identified?
2. **Check for a proof score** at `examples/patterns/` — not all patterns have them, but those that do (look for `proof_score:` in the pattern's frontmatter) are ground truth for how patterns become YAML.
3. **Check `composes_with`** — natural compositions emerge from shared forces.

Before committing to a pattern:
- Why this one and not an alternative that addresses the same forces?
- If composing: what structural property does the composition create that neither pattern has alone?
- Could any pattern be removed without losing a necessary property?
- Could the problem be served by a simpler approach with no named pattern at all? Not every problem needs orchestration patterns — sometimes sequential stages with good validations are the right answer.

### Derive the Structure

Each selected pattern has a `stages:` field describing its structural shape. Read it. For each pattern in your composition, extract a table:

| Stage | Purpose | Instrument Guidance | Depends On |
|-------|---------|---------------------|------------|

Most patterns need bookend stages beyond their core shape — setup before and consolidation after. A pattern with 4 core stages typically becomes 5-6 stages in a score. The proof scores in `examples/patterns/` show this expansion concretely.

The score's full stage list is the composition of these tables. When patterns compose, their stages interleave or sequence depending on the relationship:
- **Sequential**: One pattern's output feeds the next pattern's input. Stages chain end-to-end.
- **Nested**: One pattern's stage contains another pattern's full shape. The inner pattern replaces what would be a single stage.
- **Parallel**: Patterns operate on independent aspects simultaneously. Stages run concurrently with a shared convergence point.

Some boundaries can merge when a single stage genuinely serves both patterns' purposes — but each merge is a claim that the separation adds no value. Verify before merging.

Place a boundary where:
- The output format changes
- The instrument should change
- A validation gate is needed
- Parallelism begins or ends
- Context would exceed useful limits

Do not place a boundary where:
- Work is naturally atomic
- Splitting would lose necessary context
- The boundary exists for symmetry, not for cognition

### Select Instruments

Each pattern's stages include instrument guidance. Use it as a starting point, then consider:

- What capability does each stage actually require? Match the instrument to the task.
- Where is verification needed? Use a different model family for verification than for production — correlated models share blind spots.
- Which stages are bottlenecks — where failure wastes the most downstream work? Use the strongest available instrument there.
- Is any work deterministic? Tests, linting, structural comparison — these are CLI commands, not AI tasks.

### Design Injections

If an agent needs content to do its work, inject it. Do not tell agents to "read this file" — injection guarantees they receive it.

- **Preludes** for shared context every sheet needs: conventions, standards, glossary
- **Cadenzas** for per-sheet context: the specific input for this stage, the upstream output, per-instance data

### Design Validations

For every goal stated in the prompt, ask: **"Can the agent pass all my validations without achieving this goal?"**

If yes, the validations are decorative. Add one that directly proves the goal was achieved.

Validations prove outcomes, not process. A file existing proves only that a file was created. Tests passing proves behavior. Structural markers prove the deliverable has the shape the prompt asked for.

Layer validations coarse to fine — existence first, structure second, behavior third. If a coarse check fails, skip the fine checks.

For fan-out, validate each instance independently. For synthesis stages, validate that the output reflects integration, not concatenation.

### Self-Review

Before finalizing, step back from the pattern analysis and ask: **does this score still serve the user's actual goal?** It is possible to compose a structurally perfect score for the wrong problem. The patterns are a vocabulary, not a destination — the destination is the user's intent.

Then attack every sheet:

- Can this instrument complete this work with quality? If the answer is "probably," decompose further or strengthen the instrument.
- Is every required input injected, not just referenced?
- Can an agent pass all validations without doing the actual work?
- Are dependencies declared for all stages that need them?
- For fan-out: is `parallel.enabled: true` set? Are dependencies correct?
- Does `mzt validate` pass?
- For concerts: do all `job_path` values use absolute paths? Is `fresh: true` set for self-chaining?

---

## Workspace Safety

**Never** set workspace to the project root. If `workspace_lifecycle.archive_on_fresh` triggers, the workspace directory is archived or wiped. If workspace IS the project root, the entire project is destroyed.

**Default:** `./workspaces/{job-name}-workspace`

**For scores that modify source code**, use a dual-path pattern: workspace for artifacts and state, a `project_root` variable for the codebase. Agents work in `project_root` and write artifacts to `workspace`.

**Always:**
- Confirm the workspace path with the user
- Verify the path doesn't contain `.git/`, `pyproject.toml`, or other project root markers at its root
- Use `{{ workspace }}` in templates, `{workspace}` in validations — mixing these is the most common syntax error

---

## References

| Purpose | Where |
|---------|-------|
| YAML syntax, validation types, config structure, common pitfalls | Invoke `mozart:score-authoring` |
| Running, monitoring, debugging, recovering jobs | Invoke `mozart:usage` |
| All 56 patterns by scale, with problems and signals | `scores/rosetta-corpus/INDEX.md` |
| Individual pattern shapes, stages, composition edges | `scores/rosetta-corpus/patterns/<name>.md` |
| Forces → generators → patterns mapping | `scores/rosetta-corpus/forces.md` |
| Problem type → pattern composition mapping | `scores/rosetta-corpus/selection-guide.md` |
| Marianne terminology | `scores/rosetta-corpus/glossary.md` |
| Proof scores (how patterns become YAML) | `examples/patterns/` |
| Score archetypes and real examples | `examples/` |
| Complete score YAML reference | `docs/score-writing-guide.md` |
