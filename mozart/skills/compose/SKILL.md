---
name: compose
description: Use when the user wants to compose a Mozart score from a goal, create an orchestration config, or turn an idea into an executable multi-stage AI workflow. Triggered by /mozart:compose or when the user describes work that would benefit from Mozart orchestration.
---

# Mozart Compose

Turn goals into executable Mozart scores. One workflow: understand, brainstorm, design, compose, validate, offer to run.

A score IS the plan AND the implementation. No intermediary design docs. No multi-skill chains. The score describes the work, the conductor executes it, validations verify it.

## The Workflow

```
UNDERSTAND → BRAINSTORM → DESIGN GATE → COMPOSE → VALIDATE → OFFER
```

Follow these phases in order. Do not skip the design gate.

## Phase 1: Understand

**Dispatch agents in parallel:**

1. **Always dispatch `venue-explorer`** — reads `.mozart/spec/`, CLAUDE.md, project structure. Returns a venue profile. Uses cached profile when available.

2. **Dispatch `codebase-analyzer` IF the goal involves project-specific work** (code changes, feature additions, bug fixes, investigations touching the codebase). Skip for non-code goals (writing, research, creative work, planning).

Pass the user's goal to `codebase-analyzer` so it knows what to investigate.

**While agents run**, read the user's goal and start forming your understanding:
- What did they say they want?
- What do they likely need but didn't say? Attend to the COMP↔CULT interface: what problem are they actually solving? What would a complete solution look like?
- What complexity tier does this fall into?

**Complexity tiers** (match the score architecture to the task):

| Tier | When | Pattern |
|------|------|---------|
| Simple Pipeline | Single focused change, linear work | 3-5 sequential stages |
| Fan-Out + Synthesis | Multi-perspective analysis, creative exploration, review | Setup → N parallel lenses → synthesis |
| Concert Chain | Large feature with phases, multi-score workflows | Score A → Score B → Score C via on_success |
| Self-Chain | Continuous improvement, iterative refinement | Score chains to itself with fresh: true |
| Complex Investigation | Deep multi-phase work with nested patterns | Preprocessing + fan-out + synthesis + review |

**When agents return**, integrate their findings into your understanding. The venue profile tells you what conventions the score must respect. The codebase analysis tells you what files, patterns, and tests are involved.

## Phase 2: Brainstorm (Adaptive)

**If the goal is clear AND venue context is rich**: skip to a brief design summary and go straight to the design gate. Don't ask questions for the sake of asking.

**If ambiguous or incomplete**: ask 1-4 questions to clarify scope, constraints, and success criteria. Prefer multiple choice when possible. One question per message.

**Throughout brainstorming, attend to what the user hasn't considered:**
- COMP↔CULT: Who will use the output? What problem does this actually solve?
- COMP↔EXP: Does the proposed scope feel right, or is it a workaround for a deeper issue?
- SCI↔CULT: What does the codebase evidence suggest about how this should be approached?

Don't just do as told. If you can see that the user's goal is incomplete, that they're missing a validation strategy, that the work decomposes differently than they imagine — say so. Cover their gaps.

## Phase 3: Design Gate

**Present the score architecture before generating anything.** This is non-negotiable, but scales to complexity:

**For simple tasks** (1-2 sentences):
> "I'll compose a 3-stage pipeline: research the area → implement the change → verify tests pass. Sound good?"

**For complex tasks** (structured breakdown):
> Show: pattern choice, stage breakdown with purposes, where fan-out applies and why, what validations will verify, whether this is one score or a concert.

**Wait for user approval before composing.**

If the user pushes back or suggests changes, incorporate them. The design gate exists to align before investing in YAML generation.

## Phase 4: Compose

Generate the score YAML. Reference the bundled score-authoring doc at `${CLAUDE_PLUGIN_ROOT}/docs/score-authoring.md` for syntax rules and patterns. Reference the curated examples at `${CLAUDE_PLUGIN_ROOT}/docs/examples/` for architectural patterns.

### Composition Rules

**Syntax — the critical distinction:**
- Jinja `{{ }}` in prompt templates, prelude/cadenza file paths, capture_files
- Python `{}` in validation paths, commands, working_directory
- Template blocks use `|` (literal), never `>` (folded)
- Regex in YAML uses `\\d` (double-escaped) or single-quoted strings

**Validations — outcome-focused, always:**
- Every sheet gets at least one validation
- Never file-exists alone — combine with content checks or command checks
- Apply the litmus test: "Can the agent pass all validations without achieving the stated goal?" If yes, strengthen the validation.
- For code tasks: `command_succeeds` with test/lint/build commands
- For writing tasks: content_contains for structural markers + command_succeeds with `wc -w` for substance
- For creative/research: content_regex for expected sections + word count minimums

**Conservative defaults — first-run success:**
- `skip_permissions: true` — always
- `disable_mcp: true` — unless MCP is explicitly needed
- NO cost limits or circuit breakers — these trip up first runs
- Generous timeouts (1800s+ for anything involving builds/tests)
- Stale detection >= 1800s for verification stages
- NO skip_when expressions unless genuinely needed

**Architecture patterns:**
- `sheet.size: 1` for stage-based scores (always)
- `total_items` = number of stages (pre-expansion)
- Fan-out stages need `dependencies`, `parallel.enabled: true`, `parallel.max_concurrent`
- Cross-sheet context: `auto_capture_stdout: true`, appropriate `lookback_sheets`
- Data-driven fan-out via `prompt.variables` lookup dicts (not hardcoded per-instance prompts)
- Synthesis stages ask for convergence, tensions, emergence — never summary

**Prompts — outcome-focused, not prescriptive:**
- Describe what should be different when the sheet completes
- Give enough context to act without guessing
- Include output file paths explicitly
- Reference previous stage outputs where relevant
- For fan-out: per-instance context via `{{ lenses[instance] }}` or similar lookup pattern

**Workspace and paths:**
- Workspace paths should be absolute or clearly relative to project root
- `on_success` job_paths must be absolute
- Use `{{ workspace }}` in templates, `{workspace}` in validations

**Concert/chaining:**
- Self-chains: `fresh: true`, `max_chain_depth` set, `archive_on_fresh: true`
- Cross-score chains: absolute job_paths, explicit workspace configuration
- `detached: true` for autonomous chains, `false` for sequential pipelines

**What NOT to include unless explicitly needed:**
- `cost_limits` — kills runs
- `circuit_breaker` — kills runs
- `stale_detection` with tight timeouts — kills verification stages
- Complex `skip_when` expressions — hard to debug
- `isolation` (worktree) — only when parallel sheets modify the same repo

### Multi-Score Composition

If the goal requires a concert (multiple scores chaining together):
- Generate each score as a separate YAML file
- Wire them via `on_success: run_job` hooks
- Use absolute paths for `job_path`
- Configure workspace lifecycle for each score
- Present them as a coherent set with execution order documented

### Save Location

Infer from context:
- Scores for the user's ongoing development → `scores-internal/` (gitignored)
- Scores that are part of the project's operation → `scores/` (tracked)
- Example scores for documentation → `examples/` (tracked)
- If unclear, ask the user

## Phase 5: Validate

**Dispatch `score-reviewer` agent** with the generated score(s) and the original goal.

The reviewer is adversarial — it assumes the score has problems. It checks:
- Syntax correctness (Jinja vs format string placement)
- Validation quality (the litmus test on every sheet)
- Architecture fit (right pattern for the complexity)
- Prompt quality (enough context, outcome-focused)
- First-run safety (no features that kill runs)
- Concert/chaining correctness (if applicable)

**When the reviewer returns:**
- Fix all Critical issues immediately
- Fix Important issues before presenting to user
- Note Suggestions and apply if they improve the score meaningfully
- Do not present a score the reviewer flagged as "Not ready to run"

## Phase 6: Offer

Present the final score to the user. Show:
- The score YAML (or summary for very long scores)
- Where it will be saved
- What it will do when run (stage breakdown)

Then offer options:

**If Mozart is available** (check: `which mozart` or `mozart conductor-status`):
- "Save and run now" — save the score, submit to conductor
- "Save for later" — save the score, show the run command
- "Review first" — show the full YAML for inspection before saving

**If Mozart is not available:**
- Save the score
- Explain: "Install Mozart and start the conductor to run this score: `mozart start && mozart run <path>`"

## Score-Authoring Reference

The full score-authoring reference is bundled at `${CLAUDE_PLUGIN_ROOT}/docs/score-authoring.md`. Read it when you need syntax details, validation patterns, fan-out architecture, or Jinja techniques beyond what's covered in this skill.

## Curated Examples

Five curated example scores at `${CLAUDE_PLUGIN_ROOT}/docs/examples/`:

| File | Pattern | Teaches |
|------|---------|---------|
| `tier1-pipeline.yaml` | Sequential stages | Basic structure, proper validations, workspace conventions |
| `tier2-fanout.yaml` | Fan-out + synthesis | Data-driven parallel cognition, synthesis for emergence, non-code domain |
| `tier3-concert-chain.yaml` | Multi-score chain | Concert orchestration, on_success hooks, workspace lifecycle |
| `tier4-self-chain.yaml` | Self-chaining loop | Continuous improvement, fresh flag, chain depth, archival |
| `tier5-investigation.yaml` | Complex multi-phase | Preprocessing, nested fan-out, TDF review, Jinja macros |

Read the example closest to the user's goal before composing. Match their patterns.

## What This Skill Is NOT

- It does not write implementation plans (the score IS the plan)
- It does not execute scores (the conductor does that)
- It does not modify code (the score's agents do that when run)
- It does not replace `mozart compose` CLI (it's the Claude Code front-end for the same capability)
