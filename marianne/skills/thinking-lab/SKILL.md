---
name: thinking-lab
description: Use when you need independent expert reviews from multiple frontier models. Drop a prompt + context into the input directory, run the score, read three independent reviews, synthesize yourself.
---

# Thinking Lab Skill

> **Purpose**: 3 parallel independent reviews from different frontier model families. You synthesize with full context.

---

## Triggers

| Use This Skill | Skip This Skill |
|---|---|
| Design/architecture decisions needing diverse perspectives | Simple questions with known answers |
| Code reviews wanting independent analysis | Execution tasks (models write reviews, not code) |
| Trade-off analysis, "run this by the team" | Time-critical work (takes ~10-15 min) |

---

## Usage

### 1. Drop files into the input directory

```
scores/prep/thinking-lab-input/
├── prompt.md          ← your question (REQUIRED)
└── [any other files]  ← context, code to review, specs, etc.
```

Everything gets injected into each reviewer's context via directory cadenza.

### 2. Run

```bash
mzt run scores/prep/thinking-lab.yaml --fresh
```

### 3. Read reviews

```
workspaces/thinking-lab/review-1.md  # Opus 4.7
workspaces/thinking-lab/review-2.md  # Gemma 4
workspaces/thinking-lab/review-3.md  # GLM 5.1
```

### 4. Synthesize yourself

Look for consensus (all 3), majority (2/3), unique insights (1 only), and conflicts. You have the full conversation context — external synthesis loses it.

---

## Collaborative Build Pattern

After a review round, put all three reviews back into the input directory alongside the original, and ask them to build the integrated solution. Each model sees everyone else's feedback and produces a complete implementation.

---

## Notes

- Always use `--fresh` for new questions
- Don't put secrets in the input dir (3 providers see it)
- Typical runtime: 5-15 minutes
- Cost: ~$0.10-0.50 per run (Opus dominates)
- Models configurable in `scores/prep/thinking-lab.yaml` under `instruments:`
- Add more reviewers by adding sheets + instruments to the score

> *Many minds. One question. The divergence is the value.*
