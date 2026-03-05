---
name: composing
description: Use when the user wants to compose a Mozart score from a goal, create an orchestration config, or turn an idea into an executable multi-stage AI workflow. Triggered by /mozart:compose or when the user describes work that would benefit from Mozart orchestration.
---

# Mozart Compose

Turn goals into executable Mozart scores. One workflow: understand, brainstorm, design, compose, validate, offer to run.

A score IS the plan AND the implementation. No intermediary design docs. No multi-skill chains. The score describes the work, the conductor executes it, validations verify it.

## Principle Zero: The User Is the Authority

User directives about process supersede this workflow. If the user says "ask me questions first," enter question mode immediately — do not evaluate whether questions are needed. If the user expresses frustration or confusion, STOP the workflow and address what they're telling you. The phases below are your default structure, not a rigid pipeline that overrides what the user is communicating.

## Workspace Safety

**NEVER set workspace to the project root.** If `workspace_lifecycle.archive_on_fresh` triggers, the workspace is archived or wiped. If workspace = project root, the entire project is destroyed.

**Default pattern:** `./workspaces/{job-name}-workspace` (relative to project) or an absolute path to a dedicated directory.

**Reject these as workspace paths:**
- Any path containing `.git/`, `package.json`, `pyproject.toml`, `Cargo.toml`, `go.mod`, or `Makefile` at its root
- The project root itself
- Any path that is a parent of source code

**For scores that modify source code** (not just produce reports), use the dual-path pattern:
```yaml
workspace: "./workspaces/my-score-workspace"  # Artifacts, reports, state
prompt:
  variables:
    project_root: "/absolute/path/to/project"  # Code lives here, referenced in prompts
```

Agents work in `project_root` (via `backend.working_directory` or prompt instructions) and write artifacts to `workspace`. Validations that check project files use `command_succeeds` with hardcoded paths since user variables are not available in validation fields.

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
- What do they likely need but didn't say? What problem are they actually solving? What would a complete solution look like?
- What complexity tier does this fall into?

**Complexity tiers** (match the score architecture to the task):

| Tier | When | Pattern | Example |
|------|------|---------|---------|
| Simple Pipeline | Single focused change, linear work | 3-5 sequential stages | `tier1-pipeline.yaml` |
| Fan-Out + Synthesis | Multi-perspective analysis, creative exploration, review | Setup → N parallel lenses → synthesis | `tier2-fanout.yaml` |
| Concert Chain | Large feature with phases, multi-score workflows | Score A → Score B → Score C via on_success | `tier3-concert-chain.yaml` |
| Self-Chain | Continuous improvement, iterative refinement | Score chains to itself with fresh: true | `tier4-self-chain.yaml` |
| Complex Investigation | Deep multi-phase work with nested patterns | Preprocessing + fan-out + synthesis + review | `tier5-investigation.yaml` |

**Read the matching example.** Once you identify the complexity tier, read the corresponding example from `${CLAUDE_PLUGIN_ROOT}/docs/examples/`. This is mandatory — the examples encode proven structural patterns. Match their architecture.

**When agents return**, extract and integrate:

From **venue-explorer**:
- Conventions the score must respect (testing framework, style rules, architecture patterns)
- Constraints that affect score design (must/must-not rules)
- Stack details that inform backend and tool choices

From **codebase-analyzer** (if dispatched):
- Specific file paths and functions the score's agents will need to work with
- Test commands and infrastructure
- Integration points and risk areas
- Recommended score architecture based on code structure

Carry these findings into the design gate — they inform workspace choice, validation strategy, and prompt content.

## Phase 2: Brainstorm

**Default: Ask at least one question.** Even when the goal seems clear, confirm your understanding with the user. Good questions reveal misalignment before it becomes wasted YAML.

Questions to consider (pick the most valuable 1-3):
- **Scope**: "Should this cover X, or is Y sufficient?"
- **Success criteria**: "How will you know this worked? What does the output look like?"
- **Validation strategy**: "Should I validate with tests, structural checks, or both?"
- **Workspace**: "This will create artifacts in `./workspaces/{name}-workspace` — does that work?"
- **Model/backend**: "Standard Claude CLI, or do you need a specific model?"

**Skip questions ONLY if the user explicitly says** "just do it," "don't ask, compose," or equivalent. The user's permission to skip is required — do not self-assess that questions are unnecessary.

Prefer multiple choice when possible. One question per message.

**Throughout brainstorming, attend to what the user hasn't considered.** If you can see that the goal is incomplete, that they're missing a validation strategy, that the work decomposes differently than they imagine — say so. Cover their gaps.

## Phase 3: Design Gate

**Present the score architecture before generating anything.** This is non-negotiable.

**Every design gate must include:**
1. **Pattern**: Which tier/pattern and why
2. **Workspace**: The exact workspace path (so the user can verify it's safe)
3. **Stage breakdown**: What each stage does
4. **Validation approach**: How you'll verify outcomes (not just "validations will check")
5. **Model**: Which backend/model (if non-default)

**For simple tasks** (compact format):
> "3-stage pipeline (tier 1): research → implement → verify. Workspace: `./workspaces/input-validation-workspace`. Validations: test execution + code structure checks. Sound good?"

**For complex tasks** (structured breakdown):
> Show: pattern choice with reasoning, stage breakdown with purposes, where fan-out applies and why, specific validation strategy per stage, workspace path, whether this is one score or a concert, any known risks.

**Wait for user approval before composing.**

If the user pushes back or suggests changes, incorporate them. The design gate exists to align before investing in YAML generation.

## Phase 4: Compose

Generate the score YAML. Load the appropriate score-authoring reference docs from `${CLAUDE_PLUGIN_ROOT}/docs/ref/` based on the complexity tier identified in Phase 1:

- **Tier 1** (simple pipeline): Read `essentials.md`
- **Tier 2** (fan-out + synthesis): Read `essentials.md` + `patterns.md`
- **Tiers 3-5** (concert, self-chain, complex): Read `essentials.md` + `patterns.md` + `advanced.md`

Reference the curated examples at `${CLAUDE_PLUGIN_ROOT}/docs/examples/` for architectural patterns.

### Composition Rules

**Syntax — the critical distinction:**
- Jinja `{{ }}` in prompt templates, prelude/cadenza file paths, capture_files
- Python `{}` in validation paths, commands, working_directory
- Template blocks use `|` (literal), never `>` (folded)
- Regex in YAML uses `\\d` (double-escaped) or single-quoted strings

**Validations — outcome-focused, always:**
- Every sheet gets at least one validation
- Never file-exists alone — combine with content checks or command checks
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
- Workspace: always a dedicated directory, NEVER the project root (see Workspace Safety above)
- Default: `./workspaces/{job-name}-workspace`
- `on_success` job_paths must be absolute
- Use `{{ workspace }}` in templates, `{workspace}` in validations

**Cross-sheet context sizing:**
- `max_output_chars`: default to 10000 (not the engine default of 2000)
- Test/build output: 15000+ (pytest output is verbose)
- Prose/reports: 5000 is usually sufficient
- Structured data (JSON, YAML): 3000 is usually sufficient
- When in doubt, use `capture_files` for large structured output instead of stdout

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

### Goal-Validation Mapping

Before finalizing validations, build a mapping table:

| Prompt Goal | Validation(s) | Can Agent Bypass? |
|-------------|---------------|-------------------|
| "Add input validation" | `command_succeeds: pytest tests/test_validation.py` | No — tests must actually pass |
| "Write research findings" | `file_exists` + `content_contains "## Findings"` + `wc -w >= 200` | Partially — could write shallow content |

For every "Can Agent Bypass? = Yes" entry, add a stronger validation. This is the procedural enforcement of the litmus test. If you cannot write a validation that prevents bypass for a given goal, note it as a known gap.

### Validation Recipes by Score Type

**Code-change scores:**
- L1: `file_exists` for output artifacts
- L2: `content_contains` for structural markers in reports
- L3: `command_succeeds` with lint/type-check commands
- L4: `command_succeeds` with test execution (the actual test command, hardcoded — user variables are not available in validation fields)
- L5: `command_succeeds` with goal-specific verification (e.g., `grep -rn` to confirm a pattern was removed)

**Research/analysis scores:**
- L1: `file_exists` for each output document
- L2: `content_contains` or `content_regex` for required sections (separate checks per section, not OR-matches)
- L3: `command_succeeds` with `wc -w` for minimum substance (200+ for short docs, 500+ for analysis, 800+ for synthesis)
- L4: `content_regex` for evidence of actual analysis (citations, file paths, specific findings)

**Creative/writing scores:**
- L1: `file_exists`
- L2: `content_contains` for structural markers
- L3: `command_succeeds` with `wc -w` for substance
- L4: `content_regex` for genre-specific markers (dialogue tags for fiction, citations for academic, etc.)

**Verification/review scores:**
- L1: `file_exists` for the review document
- L2: `content_contains` for verdict/assessment section
- L3: `command_succeeds` with `wc -w` (reviews should be substantive, not rubber stamps)
- L4: `content_regex` for evidence of critical engagement (problems found, recommendations, specific references)

### Non-Claude Backend Verification

When using non-Claude backends (Ollama, Qwen, etc.), verify the model name before offering the score:
- Ollama: run `ollama list` and confirm the model name matches exactly
- Include the verification result in the confidence report

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

**Dispatch `score-reviewer` agent** with:
- The generated score YAML
- The original user goal
- The venue profile (from venue-explorer)
- The codebase analysis (from codebase-analyzer, if available)
- The design gate summary (pattern, workspace, validation strategy)

The reviewer is adversarial — it assumes the score has problems. It checks:
- Workspace safety (is workspace separate from project root?)
- Syntax correctness (Jinja vs format string placement)
- Validation quality (the litmus test on every sheet, density metrics)
- Architecture fit (right pattern for the complexity)
- Prompt quality (enough context, outcome-focused)
- First-run safety (no features that kill runs)
- Concert/chaining correctness (if applicable)

**When the reviewer returns:**
- Fix all Critical issues immediately
- Fix Important issues before presenting to user
- Note Suggestions and apply if they improve the score meaningfully
- Do not present a score the reviewer flagged as "Not ready to run"

**Confidence report** — before offering the score, summarize for the user:
- Validation quality score (from reviewer)
- Total validations: N (expected: M+ for this tier)
- Goals with outcome validations: X of Y
- Issues found and fixed: list
- Remaining suggestions: list
- Workspace: confirmed safe / WARNING

## Phase 6: Offer

Present the final score to the user. Show:
- The score YAML (or summary for very long scores)
- Where it will be saved
- What it will do when run (stage breakdown)
- The confidence report from Phase 5

Then offer options:

**If Mozart is available** (check: `which mozart` or `mozart conductor-status`):
- "Save and run now" — save the score, submit to conductor
- "Save for later" — save the score, show the run command
- "Review first" — show the full YAML for inspection before saving

**If Mozart is not available:**
- Save the score
- Explain: "Install Mozart and start the conductor to run this score: `mozart start && mozart run <path>`"

### When Things Go Wrong

If a score needs to be stopped, fixed, or restarted mid-run:

**Cancel a running job** (safe — stops the job, preserves state):
```
mozart cancel <job-name>
```
Do NOT use `mozart stop` — that kills the entire conductor, orphaning all jobs.

**Fix config and continue** (resume from where it stopped):
```
mozart resume <job-name> --reload-config -c fixed-score.yaml
```
Do NOT use `--fresh` — that wipes all completed work and starts over. `--fresh` is only for intentionally starting a brand new run (e.g., self-chaining scores).

**Common mid-run issues:**
- Wrong workspace path → cancel, fix YAML, resume
- Wrong model name → cancel, fix YAML, resume
- Validation too strict → cancel, relax validation, resume (completed sheets are kept)
- Validation too weak → let it finish, then edit and re-run with `--fresh` (you want a clean run with proper validations)

## Score-Authoring Reference

The score-authoring reference is split into three tiered docs at `${CLAUDE_PLUGIN_ROOT}/docs/ref/`:

- **`essentials.md`** — syntax, core variables, validations, config, YAML gotchas, pitfalls, pre-flight checklist (needed for ALL scores)
- **`patterns.md`** — design philosophy, fan-out, Jinja mastery, prompt engineering, cross-sheet/parallel config (tier 2+)
- **`advanced.md`** — retry/rate limiting, concert mode, post-success hooks, isolation, stale detection (tier 3+)

Load the docs matching the score's complexity tier during Phase 4. Invoke the `score-authoring` skill directly when you need all three loaded at once.

## Curated Examples

Five curated example scores at `${CLAUDE_PLUGIN_ROOT}/docs/examples/`:

| Tier | File | Pattern | Teaches |
|------|------|---------|---------|
| 1 | `tier1-pipeline.yaml` | Sequential stages | Basic structure, outcome validations, workspace conventions |
| 2 | `tier2-fanout.yaml` | Fan-out + synthesis | Data-driven parallel cognition, synthesis for emergence |
| 3 | `tier3-concert-chain.yaml` | Multi-score chain | Concert orchestration, on_success hooks, workspace lifecycle |
| 4 | `tier4-self-chain.yaml` | Self-chaining loop | Continuous improvement, fresh flag, chain depth, archival |
| 5 | `tier5-investigation.yaml` | Complex multi-phase | Preprocessing, nested fan-out, adversarial review, Jinja macros |

Read the example matching the user's complexity tier before composing. Match their patterns.

## What This Skill Is NOT

- It does not write implementation plans (the score IS the plan)
- It does not execute scores (the conductor does that)
- It does not modify code (the score's agents do that when run)
- It does not replace `mozart compose` CLI (it's the Claude Code front-end for the same capability)
