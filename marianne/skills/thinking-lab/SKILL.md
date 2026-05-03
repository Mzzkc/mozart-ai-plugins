---
name: thinking-lab
description: Use when you need independent expert reviews from multiple frontier models. Drop a prompt + context into the input directory, run the score, read five independent reviews, synthesize yourself.
---

# Thinking Lab Skill

> **Purpose**: 5 parallel independent reviews from different frontier model families. You synthesize with full context.

---

## Triggers

| Use This Skill | Skip This Skill |
|---|---|
| Design/architecture decisions needing diverse perspectives | Simple questions with known answers |
| Code reviews wanting independent analysis | Execution tasks (models write reviews, not code) |
| Trade-off analysis, "run this by the team" | Time-critical work (takes ~5-15 min) |
| Building or refreshing the instrument catalog (cross-model rating) | |

---

## The Score

`scores/prep/thinking-lab.yaml` — five parallel reviewers with directory-cadenza injection of the input dir. Pattern: `cli` stage 1 stages a setup marker; stages 2–6 fan in parallel, each routed to a different vendor family.

**Reviewers (current lineup):**

| # | Model | Vendor | Instrument |
|---|---|---|---|
| 1 | Claude Opus 4.7 | Anthropic | `claude-code` |
| 2 | Gemini 3.1 Pro | Google | `gemini-cli` |
| 3 | Gemma 4 | Google (free OpenRouter) | `opencode` |
| 4 | GLM 5.1 | Z.AI | `opencode` |
| 5 | GPT-5.5 | OpenAI | `codex-cli` |

To change the lineup, edit `instruments:` and `movements:` in the score.

---

## Usage

### 1. Drop files into the input directory

```
scores/prep/thinking-lab-input/
├── prompt.md          ← your question (REQUIRED)
└── [any other files]  ← context, code to review, specs, etc.
```

Everything gets injected into each reviewer's context via directory cadenza — no orchestration overhead, no LLM framing stage. Reviewers see the prompt and context directly.

### 2. Run

```bash
mzt run scores/prep/thinking-lab.yaml --fresh
```

### 3. Read reviews

```
workspaces/thinking-lab/review-1.md  # Opus 4.7
workspaces/thinking-lab/review-2.md  # Gemini 3.1 Pro
workspaces/thinking-lab/review-3.md  # Gemma 4
workspaces/thinking-lab/review-4.md  # GLM 5.1
workspaces/thinking-lab/review-5.md  # GPT-5.5
```

### 4. Synthesize yourself

Look for consensus (all 5), strong majority (4/5), majority (3/5), unique insights (1 only), and conflicts. You have the full conversation context — external synthesis loses it. For automated synthesis (e.g., catalog refresh), call this from another score that reads the five reviews after the lab completes.

---

## Collaborative Build Pattern

After a review round, put all five reviews back into the input directory alongside the original prompt, and ask the next round to build the integrated solution. Each model sees everyone else's feedback and produces a complete implementation. The divergence becomes a forcing function for synthesis.

---

## Use as a Sub-Score

Other scores can chain to thinking-lab via `on_success`:

```yaml
on_success:
  - type: run_job
    job_path: "/home/emzi/Projects/marianne-ai-compose/scores/prep/thinking-lab.yaml"
    detached: true
    fresh: true
```

The calling score generates `prompt.md` and any context files into `scores/prep/thinking-lab-input/` before chaining. To chain *back* to a downstream stage after the lab completes, generate a per-run wrapper of thinking-lab.yaml that adds `on_success` pointing to your follow-up score. See `scores/instrument-catalog-build.yaml` for the wrapper-generation pattern.

---

## Notes

- Always use `--fresh` for new questions (the score sets `archive_on_fresh: true` so previous outputs archive automatically).
- Don't put secrets in the input dir — five providers see it.
- Typical runtime: 5–15 minutes.
- Cost: ~$0.20–0.80 per run (Opus + GPT-5.5 dominate; Gemma + GLM are free).
- Reviewer lineup is configurable in `scores/prep/thinking-lab.yaml` under `instruments:` and `movements:`.
- Adding a 6th reviewer = add an instrument, add a movement, add a stage cadenza, add a stage validation.

> *Many minds. One question. The divergence is the value.*
