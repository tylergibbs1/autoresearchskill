# research.toml Reference

## Required Fields

| Field | Description |
|-------|-------------|
| `tag` | Run name for branch `autoresearch/<tag>` |
| `modifiable_files` | Files the agent can edit (glob patterns OK) |
| `readonly_files` | Context files the agent must NOT edit |
| `run_command` | Command to run each experiment |
| `metric_grep` | Grep pattern to extract metric from run.log |
| `metric_name` | Human-readable metric name |
| `metric_direction` | `"lower"` or `"higher"` |

## Optional Fields

| Field | Default | Description |
|-------|---------|-------------|
| `secondary_metrics` | `[]` | Additional grep patterns for secondary metrics |
| `timeout_seconds` | `300` | Max wall-clock seconds per run |
| `setup_check` | — | Command to verify prerequisites |
| `no_new_dependencies` | `true` | Prevent adding packages |
| `soft_constraints` | — | Text description of secondary concerns (e.g., memory) |

## Example

```toml
# research.toml
# Goal: minimize validation loss for transformer training

tag = "mar26"

modifiable_files = ["train.py"]
readonly_files = ["prepare.py", "README.md"]

run_command = "uv run train.py"

metric_grep = "^val_bpb:"
metric_name = "val_bpb"
metric_direction = "lower"

secondary_metrics = ["^peak_vram_mb:"]
timeout_seconds = 600

soft_constraints = "VRAM should not blow up dramatically"
```

## Writing Good Configs

- **Metric must be greppable** — the run command must print `metric_name: value`
  to stdout/stderr. If it doesn't, add a print statement to a modifiable file.
- **Scope modifiable_files tightly** — fewer files = more focused experiments.
- **Include rich readonly context** — README, configs, types, tests. More context
  produces better hypotheses.
- **Timeout = 2x expected runtime** — long enough to finish, short enough to
  not waste time on failures.
