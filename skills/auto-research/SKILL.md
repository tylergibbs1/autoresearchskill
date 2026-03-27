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
  version: "2.0.0"
  argument-hint: "<config-file>"
---

# Auto Research

An autonomous research agent that runs an infinite experiment loop: modify code,
run an evaluation, measure a metric, keep improvements, discard failures. Inspired
by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch).

The agent works in ANY project — ML training, compiler optimization, algorithm
tuning, performance benchmarking — anywhere you have a measurable metric and code
to tweak.

---

## Configuration

The skill requires a `research.toml` file in the project root. If one doesn't exist,
create it interactively with the user before starting the loop.

```toml
# research.toml — auto-research configuration

# A short name for this research run (used for branch naming)
# The branch autoresearch/<tag> must NOT already exist
tag = "experiment1"

# The file(s) the agent is allowed to modify (glob patterns OK)
modifiable_files = ["train.py"]

# Files the agent should read for context but NEVER modify
readonly_files = ["prepare.py", "README.md"]

# The command to run each experiment
# Output is redirected to run.log automatically — see "Context Management"
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

# Optional: prevent adding new dependencies
# no_new_dependencies = true

# Optional: a note about soft constraints (e.g., memory, binary size)
# The agent will weigh these but won't hard-reject experiments that violate them
# soft_constraints = "VRAM should not blow up dramatically. Some increase is acceptable for meaningful metric gains."
```

---

## Setup Phase

Follow these steps exactly before entering the experiment loop.

### 1. Read and validate config

Parse `research.toml` from the project root (or the path passed as an argument).
Verify all required fields are present.

### 2. Check branch doesn't exist

```bash
git branch --list "autoresearch/<tag>"
```

If the branch already exists, ask the user for a different tag. This must be a fresh run.

### 3. Create the branch

```bash
git checkout -b autoresearch/<tag>
```

### 4. Read all context files

Read every file in `readonly_files` and `modifiable_files` to build full understanding
of the codebase. This is your domain knowledge for forming hypotheses.

### 5. Run setup check (if configured)

If `setup_check` is defined, run it. If it fails, tell the user what's missing and halt.

### 6. Initialize results.tsv

Create `results.tsv` with a header row. Use **tab-separated** values (NOT commas —
commas break in descriptions).

The columns are:
```
commit	<metric_name>	<secondary_metric_1>	...	status	description
```

Add `results.tsv` and `run.log` to `.gitignore` if not already there — these must
survive git resets.

```bash
echo -e "results.tsv\nrun.log" >> .gitignore
```

### 7. Establish baseline

Run the experiment command once **without any modifications** to establish the baseline
metric. Log it to results.tsv with status `keep` and description `baseline`.

### 8. Confirm and begin

Tell the user: setup is complete, baseline is recorded, entering autonomous loop.
**This is the last time you interact with the user.** From here, you are fully autonomous.

---

## The Experiment Loop

**LOOP FOREVER — do NOT stop, do NOT ask permission to continue.**

The user may be asleep. They expect you to work indefinitely until manually stopped.
If each experiment takes ~5 minutes, you run ~12/hour, ~100 overnight.

### Step 1: Examine current state

```bash
git log --oneline -1
```

Review `results.tsv` — look at trends: what categories of changes worked? What failed?
What hasn't been tried? Re-read the modifiable files to see current code state.

### Step 2: Form a hypothesis

Based on the codebase, past results, and domain knowledge, pick ONE experimental
change to try. Each experiment should test ONE hypothesis so you know what caused
the result.

Prefer changes that are:
- **Informed by past results** — build on what worked, avoid what didn't
- **Meaningfully different** from recent experiments
- **Simple** — a small improvement with ugly complexity is NOT worth it

### Step 3: Implement the change

- Edit ONLY files matching `modifiable_files`
- Do NOT add new dependencies (unless config explicitly allows it)
- Do NOT modify files in `readonly_files`

### Step 4: Commit the change

```bash
git add <modified files only>
git commit -m "<short description of what this experiment tries>"
```

Do NOT commit results.tsv or run.log.

### Step 5: Run the experiment

**CRITICAL — redirect ALL output to run.log. Do NOT use tee. Do NOT let output
flood your context window.** If experiment output fills your context, you will lose
the ability to continue the loop.

```bash
timeout <timeout_seconds>s <run_command> > run.log 2>&1
```

The `timeout` command will kill the process if it exceeds the time limit.
If `timeout` isn't available, use the shell's built-in timeout mechanism.

### Step 6: Extract results

```bash
grep "<metric_grep>" run.log
grep "<secondary_metric_grep_1>" run.log
```

Parse the numeric value from the grep output. For example, if the grep returns
`val_bpb: 0.9932`, extract `0.9932`.

**If grep returns empty**, the run crashed. Go to Step 7.

### Step 7: Handle crashes

Read the last 50 lines of the log to diagnose:

```bash
tail -n 50 run.log
```

Use your judgment:
- **Trivial bug** (typo, missing import, shape mismatch, off-by-one): fix it, re-commit, re-run
- **Fundamentally broken idea** (OOM, impossible architecture, numerical instability): give up on this experiment
- **Do NOT spend more than 2–3 attempts** fixing a single crashed experiment

For crashes, log to results.tsv with metric value `0.000000` (or `0` for "higher is better"),
status `crash`, then revert and move on.

### Step 8: Log results

Append a row to `results.tsv`. Fields are **tab-separated**:

| Field | Format | Example |
|-------|--------|---------|
| commit | 7-char hash | `a1b2c3d` |
| metric | float, 6 decimal places | `0.993200` |
| secondary metrics | float, 1 decimal | `44.2` |
| status | `keep`, `discard`, or `crash` | `keep` |
| description | short text, no tabs | `increase LR to 0.04` |

For crashes: use `0.000000` for metric, `0.0` for secondary metrics.

**Do NOT commit results.tsv** — leave it untracked so git resets don't destroy it.

### Step 9: Keep or discard

- **Metric IMPROVED** (lower for "lower", higher for "higher"):
  → **KEEP** the commit. The branch advances.

- **Metric EQUAL or WORSE**:
  → **DISCARD**. Reset to the previous commit:
  ```bash
  git reset --hard HEAD~1
  ```
  The experimental change is gone. The branch is back where it started.

- **Simplicity exception**: If the metric is roughly EQUAL but the code is
  noticeably SIMPLER (fewer lines, removed complexity, cleaner logic), **KEEP** it.
  Removing code for equal results is a win.

- **Complexity exception**: If the metric improved by a tiny amount but the change
  adds significant ugly complexity (20+ lines of hacky code for 0.001 improvement),
  **DISCARD** it. The complexity cost isn't worth it.

### Step 10: Print progress (every 10 experiments)

Every 10 experiments, print a brief progress summary so the user can see status
if they check in:

```
=== Auto Research Progress ===
Branch: autoresearch/<tag>
Experiments: 47 total (12 kept, 32 discarded, 3 crashed)
Best <metric_name>: 0.9712 (experiment #38)
Baseline <metric_name>: 0.9979
Improvement: 2.67%
===
```

### Step 11: GOTO Step 1

---

## Context Window Management

**This is critical for long-running sessions.** You may run for hours. Protect your
context window:

1. **ALWAYS redirect output**: `> run.log 2>&1`. Never let experiment output print
   to your context.
2. **Use grep, not read**: Extract metrics with `grep`, don't read the full log.
3. **Only tail on crash**: `tail -n 50 run.log` — only when you need to diagnose.
4. **Don't re-read unchanged files**: Only re-read modifiable files when you need
   to see current state. Readonly files don't change — read them once.
5. **Keep commit messages short**: One line, under 80 chars.
6. **results.tsv is your memory**: If context gets long, results.tsv has your
   full experiment history. You can always re-read it.

---

## Research Strategies (When Stuck)

If you've run many experiments and improvements have stalled, try these strategies
in order:

1. **Review results.tsv for patterns**: Which categories of changes improved the
   metric? Which made it worse? Look for trends.

2. **Combine near-misses**: If experiment A improved metric by 0.001 (discarded)
   and experiment B improved by 0.001 (discarded), try combining A+B.

3. **Try the opposite**: If increasing X made things worse, try decreasing X.
   If adding a component didn't help, try removing an existing one.

4. **Vary magnitude**: If a change helped but only slightly, try a more aggressive
   version. If it hurt, try a milder version.

5. **Re-read the code carefully**: Often there are unexplored angles hiding in
   the implementation. Look at comments, TODOs, magic numbers, unused code paths.

6. **Read references**: If the code references papers, techniques, or algorithms,
   use that domain knowledge to form new hypotheses.

7. **Try radical changes**: If incremental tweaks have stalled, try something
   fundamentally different — a different algorithm, a different architecture,
   a completely different approach to the problem.

8. **Simplification sweep**: Try removing components one at a time. If the metric
   holds, you've found dead complexity. This is always a win.

9. **Rewind (sparingly)**: If you've gone down a path that feels wrong, you can
   `git reset --hard` back to an earlier known-good commit. But do this very
   rarely — usually it's better to keep iterating forward.

---

## Invoking the Skill

```
/auto-research                       # uses research.toml in project root
/auto-research path/to/config.toml   # uses a specific config file
```

If no config file exists, the agent will interactively create one with the user
before starting the loop.

---

## Example results.tsv

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
e5f6g7h	0.990100	43.8	keep	add weight decay scheduling
f6g7h8i	0.990100	43.5	keep	simplify LR scheduler (equal metric, simpler code)
g7h8i9j	0.989500	44.1	discard	0.0006 gain not worth 30 lines of complexity
```

---

## Tips for Writing Good Configs

- **Pick a clear metric**: The metric should be a single number printed to stdout/stderr.
  Format it as `metric_name: value` for easy grepping.
- **Keep timeout reasonable**: Long enough for the experiment to finish, short enough
  that failures don't waste too much time. 2x the expected runtime is a good default.
- **Scope modifiable_files tightly**: The fewer files the agent can touch, the more
  focused and reviewable experiments will be.
- **Include good readonly context**: The agent makes better hypotheses when it
  understands the full system, not just the part it can modify.
- **Document soft constraints**: Use the `soft_constraints` field to tell the agent
  about secondary concerns (memory, binary size, compile time) that matter but
  aren't the primary metric.
