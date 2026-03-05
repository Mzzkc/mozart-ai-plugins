# Mozart Score Advanced

> Concert mode, chaining, isolation, operational config, stale detection, and verification strategies. Load this alongside essentials.md and patterns.md for tier 4-5 scores.

---

## Config: Retry, Rate Limiting, Circuit Breaker, Cost Limits, Stale Detection

### Retry & Rate Limiting

```yaml
retry:
  max_retries: 3
  base_delay_seconds: 10
  max_delay_seconds: 3600       # 1 hour cap
  exponential_base: 2.0
  jitter: true
  max_completion_attempts: 3    # Completion prompts before full retry
  completion_threshold_percent: 50  # % passing to trigger completion mode

rate_limit:
  wait_minutes: 60
  max_waits: 24                 # 24 hours at default
```

### Other Configuration Sections

```yaml
circuit_breaker:
  enabled: true
  failure_threshold: 5          # Consecutive failures before OPEN
  recovery_timeout_seconds: 300

cost_limits:
  enabled: true
  max_cost_per_sheet: 2.00      # USD
  max_cost_per_job: 50.00

stale_detection:
  enabled: true
  idle_timeout_seconds: 1800    # 30min; see "Stale Detection" section

state_backend: sqlite           # json | sqlite (default: sqlite)
pause_between_sheets_seconds: 10
```

---

## Concert Mode (Self-Chaining)

```yaml
workspace_lifecycle:
  archive_on_fresh: true
  max_archives: 10

concert:
  enabled: true
  max_chain_depth: 5

on_success:
  - type: run_job
    job_path: "/absolute/path/to/my-score.yaml"   # MUST be absolute
    detached: true
    fresh: true                 # CRITICAL for self-chaining
```

---

## Post-Success Hooks

**Always use absolute paths for `job_path`.** Relative paths resolve from the daemon's CWD, not the score file's directory. If the conductor starts from a different directory, the chain silently breaks (file not found, hook result lost).

```yaml
on_success:
  - type: run_job               # Chain to another score
    job_path: "/home/user/project/next-job.yaml"  # Absolute!
    detached: true
    fresh: true
  - type: run_command           # Shell command
    command: "curl -X POST https://api.example.com/done"
  - type: run_script            # Script file
    command: "./deploy.sh"
```

---

## Isolation (Git Worktrees)

```yaml
isolation:
  enabled: true
  mode: worktree
  cleanup_on_success: true
  cleanup_on_failure: false     # Keep for debugging
  fallback_on_error: true
```

---

## Stale Detection and Verification Stages

Stale detection monitors stdout activity only. Child processes (pytest, mypy, ruff) run silently — if they take longer than `idle_timeout_seconds`, the entire process group is killed. This is the #1 cause of stuck verification stages.

**Fix: Either set a lenient timeout or fan-out verification into parallel instances.**

```yaml
# Option A: Lenient timeout (simpler)
stale_detection:
  enabled: true
  idle_timeout_seconds: 1800    # 30min — safe for heavy subprocesses

# Option B: Fan-out verification (more robust, parallelizes the work)
sheet:
  fan_out:
    9: 3    # Split verification into 3 parallel checks (tests / types / lint)
```

---

## Validation Timeouts

Measure first (`time pytest tests/ -x -q`), then set `timeout_seconds` to **max(measured × 1.5, 900)**. Test suites grow — leave room. Over 15 minutes? Split into targeted validations.

---

*Mozart Score Advanced --- extracted from the score-authoring reference.*
