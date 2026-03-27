---
name: auto-researching
description: >-
  Runs an autonomous experiment loop that modifies code, measures a target metric,
  keeps improvements, and discards failures. Operates indefinitely without human
  intervention. Triggers on "auto research", "run experiments", "optimize autonomously",
  "research loop", or "find improvements automatically".
license: MIT
metadata:
  author: tylergibbs
  version: "4.0.0"
  argument-hint: "[config-file]"
---

# Auto Research

Autonomous experiment loop: modify code, evaluate, measure metric, keep wins,
discard losses. Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch).

Works in any project with a measurable metric — ML training, compiler optimization,
algorithm tuning, performance benchmarking.

## Discovery Flow

If `research.json` exists, skip to [Setup Phase](#setup-phase).

Otherwise:

1. **Explore the project silently** — read README/docs, scan file structure, identify
   language/framework, find existing benchmarks or evaluation scripts

2. **Ask the user one question** (use `AskUserQuestion` if available):

   > I've explored your project. Here's what I found:
   > - [brief project summary]
   > - [existing benchmarks/metrics found]
   >
   > What would you like me to optimize? And how do I run an experiment?

3. **Clarify once if needed** — infer reasonable defaults otherwise:
   - `timeout_seconds`: 2x expected runtime, or 300
   - `metric_direction`: infer from name (loss/latency/size → lower; speed/accuracy → higher)
   - `tag`: today's date (e.g., `mar26`)
   - `no_new_dependencies`: true

4. **Write `research.json`** — see [CONFIG.md](CONFIG.md) for field reference.
   Show the user for confirmation, then proceed.

## Setup Phase

1. Parse and validate `research.json`
2. Verify `autoresearch/<tag>` branch doesn't exist — if it does, append `-2`, `-3`, etc.
3. `git checkout -b autoresearch/<tag>`
4. Read all `readonly_files` and `modifiable_files` for context
5. Run `setup_check` if configured; halt on failure
6. Add `results.tsv` and `run.log` to `.gitignore`
7. Initialize `results.tsv` with tab-separated header row
8. **Baseline run**: run the experiment unmodified, log as `keep | baseline`
9. Tell user setup is complete. **This is the last interaction.** From here, fully autonomous.

## The Experiment Loop

**LOOP FOREVER. NEVER stop. NEVER ask permission to continue.**

The user may be asleep. They expect you to run indefinitely until manually stopped.

```
1. Review results.tsv for trends. Re-read modifiable files for current state.

2. Form ONE hypothesis. Prefer changes that are:
   - Informed by past results
   - Meaningfully different from recent experiments
   - Simple — tiny gain + ugly complexity = not worth it

3. Edit ONLY modifiable_files. No new dependencies unless config allows it.

4. Git commit with short description. Do NOT commit results.tsv or run.log.

5. Run experiment: redirect ALL output to run.log (NEVER flood context).
   Kill if exceeds timeout_seconds.

6. Extract metric via grep. If empty → crashed.

7. Crashes: read last 50 lines of log. Trivial fix → retry (max 2-3 attempts).
   Broken idea → log as crash, revert, move on.

8. Append tab-separated row to results.tsv (untracked, survives git resets).

9. Keep or discard:
   - IMPROVED → keep commit, branch advances
   - EQUAL or WORSE → git reset --hard HEAD~1
   - Exception: equal metric + simpler code → keep
   - Exception: tiny gain + lots of ugly code → discard

10. Every 10 experiments, print a progress summary.

11. GOTO 1
```

## Critical Rules

- **Protect your context window** — see [CONTEXT.md](CONTEXT.md). This is the #1
  risk for long-running sessions. Always redirect output, use grep not read, only
  tail on crash.
- **results.tsv is your memory** — it survives git resets and contains full history.
  Re-read it when you need to review past experiments.
- **One change per experiment** — test one hypothesis at a time.
- **When stuck** — see [STRATEGIES.md](STRATEGIES.md) for research tactics.

## Example results.tsv

```
commit	val_bpb	memory_gb	status	description
a1b2c3d	0.997900	44.0	keep	baseline
b2c3d4e	0.993200	44.2	keep	increase LR to 0.04
c3d4e5f	1.005000	44.0	discard	switch to GeLU activation
d4e5f6g	0.000000	0.0	crash	double model width (OOM)
e5f6g7h	0.990100	43.8	keep	add weight decay scheduling
f6g7h8i	0.990100	43.5	keep	simplify LR scheduler (equal metric, simpler code)
```
