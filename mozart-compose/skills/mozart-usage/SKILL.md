---
name: mozart-usage
description: Use when the user is running, monitoring, debugging, or recovering Mozart jobs. Covers the conductor, job lifecycle, diagnostics, config reload, recovery, self-healing, concert operations, and anti-patterns that lose work. Do NOT use for writing score configs (use mozart-score-authoring instead).
---

# Mozart Usage Skill

> **Purpose**: Run, monitor, debug, and recover Mozart jobs. Covers the conductor, job lifecycle, diagnostics, config reload, recovery, and the anti-patterns that lose work.

---

## Triggers

| Use This Skill | Skip This Skill |
|---|---|
| Running/monitoring Mozart jobs | Writing new score configs (use mozart-score-authoring) |
| Debugging failed or stuck jobs | |
| Resuming interrupted jobs | |
| Understanding validation failures | |
| Conductor operations | |

---

## Conductor: The Required Foundation

**The conductor (daemon) is required for `mozart run`.** Without a running conductor, only `--dry-run` and `mozart validate` work.

### Starting the Conductor

```bash
# Foreground (development, see logs directly)
mozart start --foreground

# Background (production)
mozart start

# Detached from scripts (survives session end)
setsid mozart start &
```

### Conductor Commands

| Command | Purpose |
|---|---|
| `mozart start` | Start the conductor daemon |
| `mozart start --foreground` | Start in foreground (development) |
| `mozart start --profile dev` | Start with dev profile (debug logging, strace on) |
| `mozart start --profile intensive` | Start with intensive profile (48h timeout, high limits) |
| `mozart start --profile minimal` | Start with minimal profile (profiler + learning off) |
| `mozart stop` | Stop the conductor (**only when no jobs are running**) |
| `mozart stop --force` | Force-kill the conductor --- **NEVER with active jobs** |
| `mozart restart` | Stop and restart |
| `mozart restart --profile dev` | Restart with a profile |
| `mozart conductor-status` | Check if conductor is running |

### How Jobs Route Through the Conductor

When you run `mozart run config.yaml`, the CLI checks for a running conductor via Unix socket. If found, the job is submitted through IPC and the CLI returns. The conductor manages job lifecycle, rate limit coordination across concurrent jobs, and event routing. If no conductor is found, the command exits with an error.

**The conductor runs your jobs.** `mozart run` is a client that submits work and returns. The job continues in the daemon regardless of whether your terminal stays open.

**NEVER stop the conductor while jobs are actively running.** Killing the daemon orphans all in-flight Claude agent processes and corrupts job state --- sheets get stuck as `in_progress` with no validation or cleanup. To reload config on a running job, use `mozart modify -c new.yaml --resume --wait`. To safely stop: pause all jobs first, wait for pauses to take effect, then `mozart stop`.

---

## Job Lifecycle

### Submitting a Job

```bash
mozart run config.yaml              # Submit (routes through conductor)
mozart run config.yaml --dry-run    # Preview without running (no conductor needed)
mozart run config.yaml --fresh      # Fresh start (clears previous state)
mozart run config.yaml -s 5         # Start from sheet 5
mozart run config.yaml -w /path     # Override workspace directory
mozart run config.yaml --self-healing      # Auto-diagnose + fix on retry exhaustion
mozart run config.yaml --self-healing --yes  # Auto-confirm suggested fixes
mozart run config.yaml -j           # JSON output
```

| Option | Short | Description |
|---|---|---|
| `--dry-run` | `-n` | Preview execution (no conductor needed) |
| `--start-sheet` | `-s` | Override starting sheet |
| `--workspace` | `-w` | Override workspace directory |
| `--fresh` | | Clear previous state and start over |
| `--self-healing` | | Auto-diagnose and remediate on failure |
| `--yes` | | Auto-confirm self-healing fixes |
| `--json` | `-j` | Machine-readable output |

`mozart run` is the only command that accepts `-w`/`--workspace`. All other commands resolve job context from the conductor's registry using the job ID.

### Monitoring

```bash
mozart status my-job                # Check status
mozart status my-job --watch        # Live monitoring (refreshes every 5s)
mozart status my-job -W -i 10       # Watch with 10s interval
mozart status my-job -j             # JSON output
mozart list                         # List active jobs
mozart list --all                   # All jobs including completed/failed
mozart list --status running        # Filter by status
mozart logs my-job                  # View log entries
mozart logs my-job --follow         # Tail logs
mozart logs my-job --lines 200      # Last 200 lines
mozart logs my-job --level ERROR    # ERROR and above
mozart history my-job               # Execution history
```

| Command | Key Options |
|---|---|
| `status` | `--watch` / `-W`, `--interval` / `-i`, `--json` / `-j` |
| `list` | `--all`, `--status` / `-s`, `--limit` / `-l` |
| `logs` | `--follow` / `-F`, `--lines` / `-n`, `--level` / `-l`, `--json` / `-j` |
| `history` | `--json` / `-j` |

### Pausing

```bash
mozart pause my-job                 # Graceful pause at next sheet boundary
mozart pause my-job --wait          # Wait for acknowledgment
mozart pause my-job --wait -t 30    # Wait with 30s timeout
```

| Option | Short | Description |
|---|---|---|
| `--wait` | | Wait for pause acknowledgment |
| `--timeout` | `-t` | Wait timeout seconds (default: 60) |
| `--json` | `-j` | JSON output |

**How it works:** Creates a pause signal file in the workspace. The running job checks for this file between sheets, saves state, and transitions to PAUSED. Signal file is removed after acknowledgment.

### Resuming

```bash
mozart resume my-job                # Resume (auto-reloads config from YAML)
mozart resume my-job -c fixed.yaml  # Resume with a different config file
mozart resume my-job --no-reload    # Resume using cached snapshot
mozart resume my-job --force        # Force resume a completed job
mozart resume my-job --self-healing # Resume with self-healing
```

| Option | Short | Description |
|---|---|---|
| `--config` | `-c` | Override config file |
| `--no-reload` | | Use cached snapshot instead of auto-reloading |
| `--force` | `-f` | Resume even if completed |
| `--self-healing` | | Enable auto-diagnosis on failure |

### Modifying Running Jobs

`mozart modify` requires a new config file (`-c` is mandatory).

```bash
mozart modify my-job -c updated.yaml              # Pause and apply new config
mozart modify my-job -c updated.yaml --resume      # Pause, apply, and resume
mozart modify my-job -c updated.yaml --resume --wait  # Wait for pause before resume
```

| Option | Short | Description |
|---|---|---|
| `--config` | `-c` | New configuration file (required) |
| `--resume` | `-r` | Resume with new config after pausing |
| `--wait` | | Wait for pause acknowledgment |
| `--timeout` | `-t` | Timeout for pause acknowledgment (default: 60) |

### Registry Cleanup

```bash
mozart clear                        # Clear completed/failed/cancelled jobs
mozart clear --job my-job           # Clear specific job
mozart clear --status failed        # Clear only failed jobs
mozart clear --older-than 3600      # Clear jobs older than 1 hour
mozart clear --yes                  # Skip confirmation
```

### Other Commands

```bash
mozart validate config.yaml         # Pre-flight check (no conductor needed)
mozart diagnose my-job              # Full diagnostic report
mozart errors my-job --verbose      # Error details with stdout/stderr
mozart dashboard                    # Start web dashboard (default port 8000)
mozart patterns                     # View global learning patterns
```

---

## Debugging Protocol (Mandatory Order)

**ALWAYS follow this sequence. Do NOT skip to manual investigation.**

```bash
# 1. ALWAYS start here
mozart status my-job

# 2. If failed --- get diagnostics
mozart diagnose my-job

# 3. Error details
mozart errors my-job --verbose

# 4. Filter errors by sheet, type, or code
mozart errors my-job --sheet 3
mozart errors my-job --type rate_limit
mozart errors my-job --code E201

# 5. Include log snippets in diagnostic
mozart diagnose my-job --include-logs

# 6. THEN manual investigation if needed
```

### Understanding `mozart status` Output

- **Status**: RUNNING, COMPLETED, FAILED, PAUSED, CANCELLED
- **Validation**: Pass/Fail per sheet (a sheet can execute successfully but fail validation)
- **Sheets**: N/M completed, which failed, which skipped
- **Rate limits**: Current wait count

**Critical insight**: `exit_code=0` does NOT mean success. Only `validation_passed=true` means success.

### Common Failure Patterns

| Symptom | Likely Cause | Fix |
|---|---|---|
| File exists but "missing" | Wrong path syntax (`{{ }}` vs `{}`) | Check validation path syntax |
| Validation always passes | No validations configured, or too broad | Add meaningful validations |
| Pattern doesn't match | Regex anchors, escaping, or content changed | Test regex with `python3 -c "import re; ..."` |
| Command fails | Wrong `working_directory` or shell assumptions | Check CWD, test command manually |
| Sheet "passes" but work is bad | Validations too weak (file_exists only) | Add content checks or command validations |
| Config changes ignored | `--no-reload` used or file deleted | Config auto-reloads by default; check YAML file exists |
| Job hangs forever | Missing `skip_permissions: true` | Set in backend config |
| Chained job does nothing | Missing `fresh: true` in hook | Add `fresh: true` to self-chain hooks |

---

## Config Auto-Reload on Resume

**Mozart auto-reloads config from the original YAML file on resume.** The cached `config_snapshot` is a fallback when the file no longer exists on disk.

### Priority Order

1. Explicit `--config file.yaml` (always wins)
2. Auto-reload from stored `config_path` (default, if file exists)
3. Cached `config_snapshot` (fallback when file is gone or `--no-reload`)
4. Error (nothing available)

### Common Workflows

**Fix a config error and resume:**
```bash
# Edit the YAML file, then just resume --- auto-reloads
mozart resume my-job
```

**Use a completely different config:**
```bash
mozart resume my-job -c fixed.yaml
```

**Deterministic replay from cached snapshot:**
```bash
mozart resume my-job --no-reload
```

**Modify a running job's config:**
```bash
mozart modify my-job -c updated.yaml --resume --wait
```

---

## Recovery Procedures

### Rate Limit Recovery

Mozart auto-waits when rate limited (default: 60 minutes, up to 24 cycles).

```bash
mozart status my-job    # Shows PAUSED (rate_limited)

# If max_waits exhausted, just resume
mozart resume my-job
```

### Validation Failure Recovery

```bash
# 1. Check WHICH validation failed
mozart errors my-job --verbose

# 2a. Work complete but validation config is wrong --- fix YAML and resume
mozart resume my-job    # auto-reloads fixed config

# 2b. Work incomplete --- Mozart retries automatically
mozart resume my-job
```

### Interrupted Job Recovery

```bash
# First: always try resume
mozart resume my-job

# If resume fails with stale PID --- auto-clears since fix b474d45
mozart resume my-job    # Retrying usually works

# If job is truly stuck, force resume
mozart resume my-job --force
```

### State Corruption Recovery

```bash
# Start fresh from specific sheet
rm workspace/.mozart-state.db    # SQLite backend
mozart run job.yaml --start-sheet N
```

### `--fresh` vs `resume`

| Situation | Use |
|---|---|
| Job interrupted mid-progress | `mozart resume my-job` |
| Job failed, config needs fixing | `mozart resume my-job` (auto-reloads fixed YAML) |
| Self-chaining: completed an iteration | `--fresh` (via hook config) |
| User explicitly wants to start over | `mozart run my-score.yaml --fresh` |
| Job was cancelled or partially failed | `mozart resume my-job` (try first) |

**`--fresh` deletes checkpoint state and archives workspace artifacts.** It wipes hours of work if used on an interrupted job. When in doubt, try `resume` first.

---

## Self-Healing

```bash
mozart run job.yaml --self-healing
mozart run job.yaml --self-healing --yes    # Auto-confirm fixes
mozart resume my-job --self-healing
```

**How it works**: After all retries exhausted, diagnostic context is collected, applicable remedies identified and ranked, automatic fixes applied without prompting, suggested fixes prompt unless `--yes`.

**Built-in remedies**: create missing workspace, create parent directories, fix path separators (backslashes on Unix), suggest Jinja fixes, diagnose auth/CLI errors.

---

## Detached Execution

The conductor should be detached for long-running or unattended operation:

```bash
# CORRECT: Fully detached conductor
setsid mozart start &

# Development: foreground (stays attached to terminal)
mozart start --foreground
```

**Why setsid for the conductor?** Creates an independent session group. The conductor survives terminal close, context compaction, and session end.

**Jobs don't need setsid.** `mozart run` submits work to the conductor and returns. The job runs in the daemon regardless of your terminal session. Only the conductor itself needs to be detached.

---

## Concert/Chaining Operations

### Self-Chaining

Self-chaining scores chain into themselves via `on_success` hooks for iterative improvement:

```yaml
on_success:
  - type: run_job
    job_path: "/absolute/path/to/my-score.yaml"  # MUST be absolute
    detached: true
    fresh: true              # REQUIRED --- clears state
concert:
  enabled: true
  max_chain_depth: 10        # Safety limit
```

**`fresh: true` is mandatory.** Without it, the chained job loads COMPLETED state and does zero work.

### Monitoring Chains

```bash
mozart list --all                       # See all jobs (current and chained)
mozart status quality-continuous-3      # Check specific job in chain
mozart history quality-continuous-3     # View history of a chained job
```

---

## Error Codes

```
E0xx Execution    E4xx State
E1xx Rate Limit   E5xx Backend
E2xx Validation   E6xx Preflight
E3xx Config       E9xx Network
```

### E0xx: Execution Errors

| Code | Retry? | Meaning |
|---|---|---|
| E001 | Yes | Timeout (command exceeded time limit) |
| E002 | Yes | Killed by signal (external termination) |
| E003 | No | Crash (segfault, bus error, abort) |
| E004 | No | Interrupted by user (SIGINT/Ctrl+C) |
| E005 | No | Out of memory (OOM killer) |
| E009 | Yes | Unknown execution error |

### E1xx: Rate Limit / Capacity

| Code | Retry? | Meaning |
|---|---|---|
| E101 | Yes | API rate limit (wait ~1hr) |
| E102 | Yes | CLI rate limit (wait ~15min) |
| E103 | Yes | Capacity exceeded (wait ~5min) |

### E2xx: Validation Errors

| Code | Retry? | Meaning |
|---|---|---|
| E201 | Yes | Expected file missing |
| E202 | Yes | Content doesn't match pattern |
| E203 | Yes | Validation command failed |
| E204 | Yes | Validation timed out |
| E209 | Yes | Generic validation needed |

### E3xx: Configuration Errors

| Code | Retry? | Meaning |
|---|---|---|
| E301 | No | Invalid configuration |
| E302 | No | Missing required field |
| E303 | No | Config file not found |
| E304 | No | YAML/JSON parse error |
| E305 | No | MCP/plugin error |
| E306 | No | CLI mode mismatch |

### E4xx: State Errors

| Code | Retry? | Meaning |
|---|---|---|
| E401 | No | Checkpoint corruption |
| E402 | Yes | State load failed |
| E403 | Yes | State save failed |
| E404 | No | State version mismatch |

### E5xx: Backend Errors

| Code | Retry? | Meaning |
|---|---|---|
| E501 | Yes | Connection failed / Job not found |
| E502 | No | Auth/authorization failed / Job not in valid state |
| E503 | Yes | Invalid response / Cannot create signal |
| E504 | Yes | Backend timeout / Pause not acknowledged |
| E505 | No | Backend not found (ENOENT) / Invalid config |

### E6xx: Preflight Errors

| Code | Retry? | Meaning |
|---|---|---|
| E601 | No | Required path missing |
| E602 | No | Prompt too large |
| E603 | No | Working directory invalid |
| E604 | No | Validation setup invalid |

### E9xx: Network Errors

| Code | Retry? | Meaning |
|---|---|---|
| E901 | Yes | Connection failed/refused |
| E902 | Yes | DNS resolution failed |
| E903 | Yes | SSL/TLS error |
| E904 | Yes | Network timeout |
| E999 | Yes | Unknown error |

---

## Anti-Patterns

| Never | Why | Do Instead |
|---|---|---|
| `timeout 600 mozart run ...` | SIGKILL corrupts state | Let Mozart handle timeouts internally |
| Assume exit_code=0 is success | Validations may have failed | Check `validation_details` |
| Debug manually first | Mozart tools provide context | `status` -> `diagnose` -> `errors` |
| Kill running job (SIGKILL) | Orphans agents, corrupts state | `mozart pause` for graceful stop |
| Edit config during run | Changes ignored until reload | Pause first, then `mozart modify` |
| Use `--fresh` on interrupted jobs | Destroys checkpoint state | Try `resume` first |
| Stop conductor with active jobs | Orphans all in-flight agents | Pause all jobs first |

---

## Global Options

All commands support:

| Option | Short | Description |
|---|---|---|
| `--version` | `-V` | Show version |
| `--verbose` | `-v` | Detailed output |
| `--quiet` | `-q` | Errors only |
| `--log-level` | `-L` | DEBUG, INFO, WARNING, ERROR |
| `--log-file` | | Path for log output |
| `--log-format` | | json, console, or both |

---

## Quick Reference

```
CONDUCTOR                         DEBUGGING ORDER
---------                         ---------------
start [--foreground|--profile]    1. mozart status ...
stop [--force]                    2. mozart diagnose ...
restart [--profile]               3. mozart errors --verbose
conductor-status                  4. Manual investigation

JOBS                              ERROR CATEGORIES
----                              ----------------
run <config> [--dry-run|--fresh]  E0xx Execution
status <job> [--watch]            E1xx Rate limit
pause <job> [--wait]              E2xx Validation
resume <job> [-c|--no-reload]     E3xx Config
modify <job> -c <cfg> [--resume]  E4xx State
diagnose <job>                    E5xx Backend
errors <job> [--verbose]          E6xx Preflight
list [--all|--status]             E9xx Network
logs <job> [--follow]
history <job>                     GLOBAL OPTIONS
clear [--job|--status]            --------------
validate <config>                 -v, --verbose
dashboard [--port]                -q, --quiet
                                  -L, --log-level
                                  --log-file
                                  --log-format
```

---

*Mozart Usage Skill --- operational guide for running, monitoring, and debugging Mozart AI Compose jobs.*
