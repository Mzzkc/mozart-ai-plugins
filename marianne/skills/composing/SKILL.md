---
name: composing
description: Use when the user wants to compose a Marianne score from a goal, turn an idea into a multi-stage AI workflow, or design orchestration YAML. Triggered by /marianne:compose or when the user describes work that would benefit from multi-stage, multi-agent orchestration.
---

# Score Composition

Composition turns a goal into a score — a program that coordinates intelligences through staged work. You are the composer. The score is played by agents you will never meet on work you will never see. Every structural choice you make now shapes whether that work succeeds.

This skill covers the cognitive work of composition: understanding the goal, recognizing the forces at play, and arranging stages so the structure serves the work. The score-authoring skill covers YAML syntax. The command skill covers execution. This skill is about the shape between them.

Scope: **Marianne orchestration.** The concrete apparatus — instruments, validation types, workspace safety, `mzt` commands — assumes Marianne. The methodology (forces, decomposition, pattern composition) transfers elsewhere; the apparatus does not. If a user asks for composition outside Marianne, say so and adapt.

---

## Preflight

Composition needs corpus vocabulary in working memory. But dumping the whole corpus upfront buries the user's goal under abstract theory. Read in tiers, tied to when you actually need each file.

**Tier 1 — before you analyze forces:**

1. `scores/rosetta-corpus/INDEX.md` — all 56 patterns, by scale, with problems and signals.
2. `scores/rosetta-corpus/forces.md` — the ten forces and their generators.

**Tier 2 — just-in-time, as you select patterns:**

3. `scores/rosetta-corpus/selection-guide.md` — the section matching your problem type.
4. `scores/rosetta-corpus/patterns/<name>.md` — the full file for each candidate pattern. Not a summary. Not from memory. The file itself. Pattern files carry the shape you are about to implement.

**Tier 3 — on demand:**

- `examples/` and `examples/patterns/` — proof scores, as reference for shape.
- `scores/rosetta-corpus/glossary.md` — when terminology is unclear.

Tiered reading is not permission to skip. What it buys is **attention preservation**: the user's goal stays fresh through the moment of composition. Pattern files must be read before you claim their patterns; that is the one rule here worth being rigid about, because the rest of this skill assumes you have the shape in hand.

---

## Principle Zero

The user is the authority. You propose; they decide. Present reasoning, offer alternatives, get approval before generating YAML.

---

## Phase 0 — Decide Whether to Compose

Not every request warrants a score. Orchestration machinery pays for itself when there is real coordination; when there isn't, it's pure overhead. Answer this before anything else.

A score is the right tool when **two or more** apply:

- The work decomposes into distinct stages with different artifacts or capabilities.
- Independent units benefit from parallel execution — fan-out over files, sources, perspectives.
- Different stages want different instruments.
- Validation between stages matters — contamination downstream costs more than catching it here.
- The work is long-running, retryable, or needs recovery.
- The score will be run again, by someone else, or in different contexts.

A score is **not** the right tool when:

- The user wants a single piece of output one good prompt could produce.
- The user wants one command run with reasoning — that's a CLI invocation.
- The user is exploring and premature structure would freeze the wrong shape.
- The user has the answer and just needs it typed out.
- The score's overhead exceeds the work itself.

**At Phase 0:**

1. Say in one sentence what the user wants.
2. Say in one sentence what serves it — a score, a prompt, a command, a conversation.
3. If it's not a score, say so and recommend the simpler path. If the user wants help with the simpler path, help them with it directly — don't abandon them to navigate without you. If they want the score anyway, they get to overrule you; note the concern and proceed.
4. If it is a score, name the criteria that apply, then continue.

Over-composition is a failure mode of this skill. Under-composition is a failure mode of neglect. The gate sits between them.

---

## Vocabulary

A **score** is a YAML configuration declaring what work to do, which instruments to use, how to validate, and how to recover.

A **sheet** is one agent performing one task — the atomic unit of execution. A **stage** is a logical phase that may fan out into multiple sheets — parallel instances of the same work on different data with the same instrument. If instances need different instruments, they are separate stages, not fan-out. In YAML, stages are declared via `movements:` and `sheet.total_items`; the `stage` template variable gives each sheet its phase number.

An **instrument** is an AI backend or CLI tool with specific capabilities and cost profile. `mzt instruments list` shows what's registered.

**Preludes** inject shared context into every sheet. **Cadenzas** inject per-sheet context. Injection is how you give an agent what it needs. Telling an agent to "read a file" is unreliable; injecting the file is not.

A **concert** chains scores via `on_success`. The **workspace** is shared memory between sheets. The **conductor** is the daemon.

The musical vocabulary is load-bearing, not decoration. When a name feels forced, the architecture may be wrong.

### Forces

Every coordination problem has a force profile. Forces drive pattern selection.

| Force | The question it asks |
|-------|---------------------|
| **Information Asymmetry** | Do different agents need to know different things? |
| **Finite Resources** | Does the work exceed what one agent can handle well? |
| **Partial Failure** | Can components fail independently? |
| **Exponential Defect Cost** | Do problems found late cost more than problems found early? |
| **Producer-Consumer Mismatch** | Do stages produce in formats the next can't directly consume? |
| **Instrument-Task Fit** | Do different tasks need fundamentally different capabilities? |
| **Convergence Imperative** | Does iterative work need domain-specific stopping criteria? |
| **Accumulated Signal** | Must evidence build to a threshold before triggering change? |
| **Structured Disagreement** | Are single perspectives unreliable for this problem? |
| **Progressive Commitment** | Is full commitment before validation risky? |

`scores/rosetta-corpus/forces.md` maps forces to generators and generators to patterns.

### The Rosetta Corpus

The corpus contains 56 coordination patterns organized by scale — within-stage, score-level, concert-level, and cross-cutting. Each pattern file carries: problem, signals, forces, generators, stages (the shape), composes-with, core dynamic, failure mode, and a YAML sketch.

Patterns are structural moves, not templates. They name shapes that exist because coordination requires them. Composing multiple patterns creates a shape none of them have alone.

**The corpus is reference, not scripture.** It captures recurring shapes observed to date, not all valid shapes. When honest force analysis finds no pattern matching more than a signal or two, composing from first principles is not a fallback — it's the right move. See *Select Patterns* below.

---

## The Workflow

```
DECIDE → UNDERSTAND → QUESTION → DESIGN GATE → COMPOSE → REVIEW → OFFER
```

Phase 0 is above. The rest assume composition is right.

### Phase 1 — Understand

Study the goal:

- What did the user say they want?
- What do they probably need but didn't articulate?
- What would make the output wrong?

**Offer to investigate the project** before designing. Conventions discovered now prevent validation failures later. The user may already have the context, or the goal may not need project-specific knowledge.

Two investigation dimensions matter:

1. **Venue context** — structure, spec corpus at `.marianne/spec/`, project docs, available instruments.
2. **Codebase analysis** (code goals only) — relevant files, test commands, build/lint commands, integration points.

If your platform supports subagents, `venue-explorer` and `codebase-analyzer` run these in parallel. If not, do it yourself.

### Phase 2 — Question

Ask at least one question. Even when the goal seems clear, confirm your read. Focus on what the user hasn't considered — missing validation strategies, unclear decomposition, unstated constraints. One question per message, multiple choice where possible. Skip only if explicitly told to.

### Phase 3 — Design Gate

Present the architecture before generating YAML. This is a turn boundary, not a mid-response ritual. You produce the design; you stop; the user responds; then — and only then — do you generate YAML. If you catch yourself writing design and YAML in the same response, you have collapsed the gate.

Show:

- The forces you identified and why they are active.
- The pattern(s) you selected (or that no pattern fit, and the structure was derived from first principles — name the forces that shaped it).
- How stages decompose: roughly what each does, what depends on what.
- Which instruments serve each stage.
- Where the workspace lives.
- How validations prove the outcome.

Concise for straightforward compositions, fuller for complex ones. Wait for approval.

### Phase 4 — Compose

Generate the YAML. Before writing, load the score-authoring reference — invoke `marianne:score-authoring` or read the tiered docs in this plugin's `docs/ref/`:

- `essentials.md` — syntax, validation types, config, common pitfalls.
- `patterns.md` — fan-out, multi-stage, Jinja, prompt engineering.
- `advanced.md` — concerts, self-chaining, isolation, stale detection.

### Phase 5 — Review

Review adversarially. Dispatch `score-reviewer` if available, or review yourself assuming the score is wrong until proven otherwise. Check workspace safety, YAML correctness, validation quality, first-run safety. Fix critical and important issues before presenting.

### Phase 6 — Offer

Present the score. Show where it saves, what it does, what the review caught. If the conductor is running, offer to submit it. If not, show the run command.

Save location by context:

- Ongoing development → `scores-internal/`
- Project operations → `scores/`
- Documentation examples → `examples/`
- Unclear → ask.

---

## The Composition Methodology

This is the heart of Phase 4: how to move from approved design to correct score.

Composition is judgment. The steps below structure that judgment; they are not a rigid order. Decomposition can reveal wrong pattern selection. Force analysis can reframe the goal. Move fluidly.

### Define the Work

Before touching patterns or YAML:

- What is the input — data, files, state?
- What is the output — what must exist when the score completes?
- What are the units of work? Can they run independently?
- What makes output wrong? What distinguishes done from done well?
- What varies across units? What must be shared?
- What depends on what?

These answers determine everything that follows.

### Analyze Forces

Walk the ten forces. For each, is it active here? Note the evidence — a sentence or two on what makes it active or not. Evidence grounds pattern selection. Without it, pattern choice collapses into name recognition.

`scores/rosetta-corpus/selection-guide.md` organizes patterns by problem type. Start there when the force profile matches a known problem type.

### Select Patterns

Active forces point to candidate patterns. **For each candidate, read the pattern file** at `scores/rosetta-corpus/patterns/<name>.md`. Not a summary. Not from memory. The file itself. Pattern files carry the structural shape — stages, instrument guidance, dependency topology — that you need in the next step. Without the file open, you cannot derive structure; you can only name the pattern.

For each candidate, hold in mind:

- The pattern's `problem` statement and how well it matches yours.
- Which signals match (two or more suggests fit; fewer suggests forcing).
- The minimum stage list and what each stage does.
- The instrument guidance per stage.
- The `composes_with` list — next-round candidates.
- The `Failure Mode` section — this is how the pattern breaks when structure is degraded.

Before committing:

- Why this pattern and not an alternative addressing the same forces?
- If composing: what structural property does the composition create that neither pattern has alone?
- Could any pattern be removed without losing a necessary property?
- Could the problem be served by sequential stages with good validations and no named pattern at all?

**When no pattern fits**, compose from first principles. This is a peer path, not a fallback:

1. Derive stages from the work's dependency graph — what inputs, what outputs, what must precede what.
2. Use the forces framework to justify each boundary — why one stage ends here and the next begins there.
3. At the Design Gate, tell the user explicitly: "No Rosetta pattern matched; I derived the structure from forces and dependencies." Name the forces you applied.
4. Append one line to `docs/technique-ideas.md` describing the shape and why existing patterns didn't cover it. Novel shapes are how the corpus grows.

First-principles composition is the same work as pattern composition with a different input. You still derive stages, place boundaries, select instruments, design injections and validations. What you give up is the vocabulary that lets you borrow structural commitments from prior art. What you gain is fit.

### Derive the Structure

Walk each selected pattern's `stages:` field. For each, know the purpose, the instrument capability, and the dependencies. Compose: sometimes patterns chain sequentially (one's output feeds the next), sometimes they nest (one's single stage expands into another's full shape), sometimes they run in parallel with a convergence point.

Patterns have *core* stages — the distinctive cognitive work — and *bookend* stages like setup, ingestion, and consolidation that naturally overlap across patterns. Bookends can merge when a single stage genuinely serves both patterns. A merge is honest when the merged stage carries every validation from both source stages, every dependency from both source stages, operates on the same artifact, and uses an instrument capable of both stages' work. A merge that drops a validation, decouples a dependency, or collapses distinct artifacts isn't a merge; it is removing structure. If you can't explain what each pattern's core stages are doing in your composition, the pattern is not implemented.

Place a stage boundary where:

- The output format changes.
- The instrument should change.
- Work must be verified before it feeds downstream.
- Parallelism begins or ends.
- Context would exceed useful limits.

Don't place a boundary where:

- Work is naturally atomic.
- Splitting would lose necessary context.
- The boundary is symmetric, not cognitive.

### Select Instruments

Different instruments — and different models within them — were built for different work. A large-context model excels at reading and synthesis. A fast model excels at mechanical tasks. A reasoning model excels at design. A CLI tool excels at deterministic operations. Match each sheet to the instrument whose purpose fits the work.

Starting from each pattern's instrument guidance, ask:

- What capability does this stage actually require — reasoning depth, context window, tool access, speed?
- Where is verification needed? Verification by a different model family than production — correlated models share blind spots.
- Where are the bottlenecks, where failure wastes the most downstream work? Strongest instrument there.
- Is any work deterministic? That's a CLI, not an AI.

Every assignment needs a fallback chain (`instrument_fallbacks:`) so rate limits or outages don't halt the score. Degrade gracefully: strong → capable → any that can complete the work.

**Verify availability.** `mzt instruments list` shows what is registered on this conductor. A score referencing an unregistered instrument fails at runtime, not at `mzt validate`. Prefer instruments you verified over instruments you remembered.

### Design Injections

Every score should use preludes and cadenzas. If an agent needs content to do its work, inject it — don't tell agents to "read this file." Telling is unreliable; injection is not.

- **Preludes** — shared context for every sheet. Conventions, standards, the spec passage, the glossary. A score without preludes leaves each agent to find context independently; they will find it differently, or not at all.
- **Cadenzas** — per-sheet context for specific sheets. The pattern file for this stage, the upstream output to review, the per-instance data for fan-out specialization.

Preludes establish the substrate. Cadenzas specialize each stage.

### Design Validations

Three questions, in order:

1. **Can the agent pass all my validations without achieving this goal?** If yes, the validations are decorative. Add one that proves the goal.
2. **If the work is done but done poorly, do my validations catch that?** Completion and quality are different. A file can exist and contain nothing useful. Code can pass tests and be unmaintainable. Distinguish adequate from inadequate, not just present from absent.
3. **When a stage produces bad output, what happens?** If the answer is "it flows downstream and contaminates everything after it," the score has a structural gap. Catching problems early is itself work that belongs in the structure.

Validations prove outcomes, not process. Layer coarse to fine — existence first, structure second, behavior third. Skip fine checks when coarse checks fail.

For fan-out, validate each instance independently. For synthesis stages, validate the output reflects integration, not concatenation.

**Negative-test your validations.** Ask: if the agent wrote an empty file, a one-line placeholder, or a refusal, would this validation catch it? Every markdown artifact stage should have a word-count or structural-marker check alongside `file_exists` — `file_exists` alone is the most common decorative validation.

### Self-Review

Step back from the structural work and ask: **does this score still serve the user's goal?** Structural perfection for the wrong problem is still wrong. The patterns are vocabulary; the destination is intent.

Then, with that alignment held, consider each stage:

- Can this instrument do this work with quality? If "probably," decompose further or strengthen the instrument.
- Is every required input injected, not just referenced?
- Can the agent pass validations without doing the work?
- Are dependencies declared for everything that needs them?
- For fan-out: `parallel.enabled: true`? Dependencies right?
- Does `mzt validate` pass?
- For concerts: absolute `job_path`? `fresh: true` for self-chaining?
- Would each validation catch an empty or placeholder output?
- Are all referenced instruments registered?

Self-review is reflection, not a checklist you race through. If you are moving fast enough to not notice which stage fails a question, you are moving too fast.

---

## Workspace Safety

**Never** set workspace to the project root. If `workspace_lifecycle.archive_on_fresh` triggers, the workspace is archived or wiped. If workspace is the project root, the project is destroyed.

**Default:** `./workspaces/{job-name}-workspace`

For scores that modify source code, use a dual-path pattern: workspace for artifacts and state, a `project_root` variable for the codebase. Agents work in `project_root` and write artifacts to `workspace`.

Always:

- Confirm the workspace path with the user.
- Verify the path doesn't contain `.git/`, `pyproject.toml`, or other project-root markers at its root.
- Use `{{ workspace }}` in templates, `{workspace}` in validations — mixing these is the most common syntax error.

---

## References

| Purpose | Where |
|---------|-------|
| YAML syntax, validation types, config structure, common pitfalls | `marianne:score-authoring` |
| Running, monitoring, debugging, recovering jobs | `marianne:command` |
| All 56 patterns by scale, with problems and signals | `scores/rosetta-corpus/INDEX.md` |
| Individual pattern shapes, stages, composition edges | `scores/rosetta-corpus/patterns/<name>.md` |
| Forces → generators → patterns mapping | `scores/rosetta-corpus/forces.md` |
| Problem type → pattern composition mapping | `scores/rosetta-corpus/selection-guide.md` |
| Marianne terminology | `scores/rosetta-corpus/glossary.md` |
| Proof scores (how patterns become YAML) | `examples/patterns/` |
| Score archetypes and real examples | `examples/` |
| Complete score YAML reference | `docs/score-writing-guide.md` |
