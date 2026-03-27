---
name: auto-research
description: >-
  Autonomous research loop that iteratively experiments on code, measures results,
  keeps improvements, and discards failures. Use when asked to "auto research",
  "run experiments", "optimize this autonomously", "research loop", or
  "find improvements automatically".
license: MIT
metadata:
  author: tylergibbs
  version: "1.0.0"
  argument-hint: "<config-file>"
---

# Auto Research

An autonomous research agent that runs an infinite experiment loop: modify code,
run an evaluation, measure a metric, keep improvements, discard failures. Inspired
by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch).

The agent works in ANY project — ML training, compiler optimization, algorithm
tuning, performance benchmarking — anywhere you have a measurable metric and code
to tweak.

## Configuration

The skill requires a `research.toml` file in the project root. If one doesn't exist,
create it interactively with the user before starting the loop.

```toml
# research.toml — auto-research configuration

# A short name for this research run (used for branch naming)
tag = "experiment1"

# The file(s) the agent is allowed to modify (glob patterns OK)
modifiable_files = ["train.py"]

# Files the agent should read for context but NEVER modify
readonly_files = ["prepare.py", "README.md"]

# The command to run each experiment
# Output is redirected to run.log automatically
run_command = "uv run train.py"

# How to extract the primary metric from run.log
# This is a grep pattern applied to run.log
metric_grep = "^val_bpb:"

# Name of the primary metric (for logging)
metric_name = "val_bpb"

# Direction: "lower" means lower is better, "higher" means higher is better
metric_direction = "lower"

# Optional secondary metrics to extract and log (grep patterns)
secondary_metrics = ["^peak_vram_mb:"]

# Maximum wall-clock seconds per experiment before killing it
timeout_seconds = 600

# Optional: command to run before the loop starts (e.g., data preparation check)
# setup_check = "test -d ~/.cache/data"

# Optional: packages/dependencies that must NOT be added
# no_new_dependencies = true
```

## How It Works

### Setup Phase

1. **Read config**: Parse `research.toml` from the project root.
2. **Create branch**: Create `autoresearch/<tag>` from the current branch.
3. **Read context**: Read all files listed in `readonly_files` and `modifiable_files`
   for full context.
4. **Run setup check**: If `setup_check` is defined, run it and halt if it fails.
5. **Initialize results log**: Create `results.tsv` with header row:
   `commit | <metric_name> | status | description`
   (Plus columns for any `secondary_metrics`.)
6. **Baseline run**: Run the experiment command once without modifications to
   establish the baseline metric.

### Experiment Loop (runs indefinitely)

```
LOOP FOREVER:

1. Examine current state
   - Read the current git branch/commit
   - Review results.tsv for past experiments and trends
   - Re-read modifiable files for current code state

2. Form a hypothesis
   - Based on the codebase, past results, and domain knowledge,
     pick an experimental change to try
   - Prefer changes that are:
     a) Informed by past results (what worked, what didn't)
     b) Meaningfully different from recent experiments
     c) Simple — a small improvement with ugly complexity is not worth it

3. Implement the change
   - Edit ONLY the files matching modifiable_files
   - Do NOT add new dependencies unless config allows it
   - Do NOT modify readonly_files

4. Commit the change
   - git add <modified files>
   - git commit -m "<short description of what this experiment tries>"

5. Run the experiment
   - Execute: <run_command> > run.log 2>&1
   - If the run exceeds timeout_seconds, kill it and treat as failure

6. Extract results
   - grep metric_grep from run.log
   - grep each secondary_metrics pattern from run.log
   - If grep returns empty → the run crashed

7. Handle crashes
   - Read last 50 lines of run.log for the error
   - If it's a trivial bug (typo, missing import, shape mismatch): fix and re-run
   - If the idea is fundamentally broken: log as "crash", revert, move on
   - Don't spend more than 2-3 attempts fixing a crashed experiment

8. Log results
   - Append a row to results.tsv (do NOT commit results.tsv)
   - Fields: commit_hash, metric_value, status, description
     (plus secondary metric values)

9. Keep or discard
   - If metric IMPROVED (respecting metric_direction): KEEP the commit
   - If metric is EQUAL or WORSE: git reset --hard to the previous commit
     (discarding the experimental change)

10. GOTO 1 — NEVER stop, NEVER ask permission to continue
```

### Critical Rules

- **NEVER STOP**: Once the loop begins, do NOT pause to ask the user anything.
  The user may be away. Run indefinitely until manually interrupted.
- **NEVER modify readonly files**: They are context only.
- **results.tsv is NOT committed**: It stays untracked so git resets don't destroy it.
- **Simplicity wins**: Equal metric + simpler code = keep. Tiny improvement +
  ugly complexity = discard. Removing code for equal results = definitely keep.
- **Think harder when stuck**: Re-read the codebase, review what worked, try
  combining near-misses, try more radical changes. Do not give up.
- **One change at a time**: Each experiment should test ONE hypothesis so you
  know what caused the result.

## Invoking the Skill

```
/auto-research                    # uses research.toml in project root
/auto-research path/to/config.toml  # uses a specific config file
```

If no config file exists, the agent will interactively create one with the user
before starting the loop.

## Output

The agent produces:

1. **Git history** on `autoresearch/<tag>` — each kept improvement is a commit
2. **results.tsv** — full experiment log with metrics and descriptions
3. **run.log** — output from the most recent experiment run

Example `results.tsv`:

```
commit	val_bpb	status	description
a1b2c3d	0.9979	keep	baseline
b2c3d4e	0.9932	keep	increase learning rate to 0.04
c3d4e5f	1.0050	discard	switch to GeLU activation
d4e5f6g	0.0000	crash	double model width (OOM)
e5f6g7h	0.9901	keep	add weight decay scheduling
```

## Tips for Writing Good Configs

- **Pick a clear metric**: The metric should be a single number printed to stdout/stderr.
  Format it as `metric_name: value` for easy grepping.
- **Keep timeout reasonable**: Long enough for the experiment to finish, short enough
  that failures don't waste too much time.
- **Scope modifiable_files tightly**: The fewer files the agent can touch, the more
  focused and reviewable experiments will be.
- **Include good readonly context**: The agent makes better hypotheses when it
  understands the full system, not just the part it can modify.
