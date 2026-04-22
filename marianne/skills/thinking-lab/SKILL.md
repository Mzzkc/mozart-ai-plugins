---
name: thinking-lab
description: Use when you need independent expert reviews from multiple frontier models on a design decision, architecture question, code review, or any problem that benefits from diverse perspectives. Drop a prompt, run the score, read three independent reviews.
---

# Thinking Lab Skill

> **Purpose**: Get 3 parallel independent reviews from different frontier model families on any question. Each reviewer sees the same prompt and context but cannot see the others' responses. You synthesize with full context.

---

## Triggers

| Use This Skill | Skip This Skill |
|---|---|
| Architecture/design decisions needing diverse perspectives | Simple questions with known answers |
| Code reviews wanting independent analysis | Tasks that need execution, not review |
| Trade-off analysis where reasonable people disagree | Anything time-critical (takes ~10-15 min) |
| "Run this by the team" / "Get a second opinion" | When you already have strong conviction |
| Technique compression / prompt optimization | |
| Security review of an approach | |

---

## How It Works

The thinking lab score (`scores/prep/thinking-lab.yaml`) fans out to 3 frontier models in parallel:

| Sheet | Model | Provider | Strengths |
|-------|-------|----------|-----------|
| 2 | Opus 4.7 | Anthropic | Deep reasoning, architectural judgment |
| 3 | Gemma 4 | Google (OpenRouter) | Practical engineering, edge cases |
| 4 | GLM 5.1 | Z.AI | Systems thinking, honest critique |

Sheet 1 is a no-op setup stage (cli instrument) that gates the parallel fan-out.

---

## Usage

### 1. Write Your Prompt

Create a markdown file with your question, context, and what you want reviewed. Be specific — vague prompts produce vague reviews.

### 2. Drop Into the Input Directory

Place your prompt and any supporting files into the cadenza directory:

```
scores/prep/thinking-lab-input/
├── prompt.md          ← your question (REQUIRED)
├── context-file.md    ← any supporting material
├── code-to-review.py  ← code they should look at
└── ...                ← anything else relevant
```

Everything in this directory gets injected into each reviewer's context via directory cadenza. The reviewers see ALL files, not just the prompt.

### 3. Run the Score

```bash
mzt run scores/prep/thinking-lab.yaml --fresh
```

Use `--fresh` to archive previous results and start clean.

### 4. Monitor

```bash
mzt status thinking-lab --watch
```

Typical runtime: 5-15 minutes depending on prompt complexity and model response times.

### 5. Read the Reviews

```bash
cat workspaces/thinking-lab/review-1.md  # Opus
cat workspaces/thinking-lab/review-2.md  # Gemma
cat workspaces/thinking-lab/review-3.md  # GLM
```

### 6. Synthesize

You (the agent with full conversation context) synthesize the three reviews. Look for:

- **Consensus** (all 3 agree) — high confidence, implement
- **Majority** (2 of 3) — likely correct, investigate the dissent
- **Unique insights** (1 only) — valuable if well-reasoned, verify
- **Conflicts** — present both sides, let the user decide

---

## Patterns

### Design Review

```markdown
# prompt.md
We're considering [approach A] vs [approach B] for [problem].
Context: [what we've built, what constraints exist]
Questions:
1. Which approach and why?
2. What are we missing?
3. What would you do differently?
```

### Code Review

```markdown
# prompt.md
Review the attached code for [specific concerns].
The code is in `code.html` in this directory.
Focus on: [performance / correctness / architecture / security]
Questions:
1. What's wrong?
2. What would you refactor?
3. Performance concerns?
```

### Collaborative Build

After a review round, put all three reviews back into the input directory alongside the original code, and ask them to build the integrated solution:

```markdown
# prompt.md
You have: the original code, and three independent reviews.
Build the complete updated version incorporating the best fixes
from all three reviews. Write the full file.
```

This creates a multi-round collaboration loop where each model sees everyone else's work and builds on it.

---

## Anti-Patterns

| Don't | Why | Instead |
|-------|-----|---------|
| Ask yes/no questions | 3 models saying "yes" isn't more useful than 1 | Ask "how" and "what's missing" |
| Include synthesis stage | The calling agent has full context; external synthesis loses it | Read reviews yourself, synthesize with context |
| Use for execution tasks | The models write to review files, not your codebase | Use for analysis, then implement yourself |
| Forget `--fresh` | Old reviews from previous runs will confuse you | Always `--fresh` for new questions |
| Put secrets in the input dir | All files get injected into 3 different model providers | Scrub API keys, credentials, etc. |

---

## Configuration

The score lives at `scores/prep/thinking-lab.yaml`. To change models:

```yaml
instruments:
  reviewer-opus:
    profile: claude-code
    config:
      cli_model: claude-opus-4-7    # ← change model here
  reviewer-gemma:
    profile: opencode
    config:
      cli_model: "google/gemma-4:free"
  reviewer-glm:
    profile: opencode
    config:
      cli_model: "zai-coding-plan/glm-5.1"
```

The input directory path is configured in the `cadenzas` block. Default: `scores/prep/thinking-lab-input/`.

---

## Cost

- Opus 4.7: uses Anthropic subscription (charged per token)
- Gemma 4 free: zero cost (OpenRouter free tier)
- GLM 5.1: zero cost (Z.AI coding plan, unlimited)

Typical run: ~$0.10-0.50 depending on prompt + context size (Opus dominates cost).
