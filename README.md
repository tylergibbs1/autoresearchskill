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

Just run the skill — no config needed:

```
/auto-research
```

The agent explores your project, asks what you want to optimize, writes a `research.json` config, then loops indefinitely — trying experiments, keeping improvements, discarding failures. Leave it running overnight and wake up to results.

If you already have a `research.json`, it skips straight to the loop.

## Configuration

The agent writes this for you, but here's the format:

```json
{
  "tag": "experiment1",
  "modifiable_files": ["train.py"],
  "readonly_files": ["prepare.py", "README.md"],
  "run_command": "uv run train.py",
  "metric_grep": "^val_bpb:",
  "metric_name": "val_bpb",
  "metric_direction": "lower",
  "secondary_metrics": ["^peak_vram_mb:"],
  "timeout_seconds": 600
}
```

See `skills/auto-research/examples/` for more:
- `ml-training.json` — ML model optimization
- `performance-bench.json` — algorithm performance tuning
- `compiler-opt.json` — compiler/codegen optimization

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
