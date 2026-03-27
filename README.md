# auto-research skill

An autonomous research agent skill for Claude Code. Runs an infinite experiment loop: modify code, run evaluation, measure a metric, keep improvements, discard failures.

Inspired by [Karpathy's autoresearch](https://github.com/karpathy/autoresearch), generalized to work in any project with a measurable metric.

## Install

```bash
npx skills add tylergibbs1/autoresearchskill
```

Or install manually:

```bash
cp -r skills/auto-research ~/.claude/skills/auto-research
```

## Usage

1. Create a `research.toml` in your project root (see examples below)
2. Run the skill:

```
/auto-research
```

The agent will create a git branch, establish a baseline, then loop indefinitely — trying experiments, keeping improvements, discarding failures. Leave it running overnight and wake up to results.

## Configuration

Create `research.toml` in your project:

```toml
tag = "experiment1"
modifiable_files = ["train.py"]
readonly_files = ["prepare.py", "README.md"]
run_command = "uv run train.py"
metric_grep = "^val_bpb:"
metric_name = "val_bpb"
metric_direction = "lower"
secondary_metrics = ["^peak_vram_mb:"]
timeout_seconds = 600
```

See `skills/auto-research/examples/` for more config examples:
- `ml-training.toml` — ML model optimization
- `performance-bench.toml` — algorithm performance tuning
- `compiler-opt.toml` — compiler/codegen optimization

## How It Works

```
LOOP FOREVER:
  1. Form hypothesis based on past results
  2. Edit modifiable files
  3. Git commit
  4. Run experiment → run.log
  5. Extract metric
  6. If improved → keep commit, advance branch
  7. If worse → git reset, discard
  8. Log to results.tsv
```

## Output

- **Git history** on `autoresearch/<tag>` — each kept improvement is a commit
- **results.tsv** — full experiment log
- **run.log** — most recent experiment output
