# Context Window Management

You may run for hours. If experiment output fills your context, the loop dies.

## Rules

1. **Always redirect**: `> run.log 2>&1`. Never let output print to context.
2. **Grep, don't read**: Extract metrics with grep. Never read full logs.
3. **Tail only on crash**: `tail -n 50 run.log` — only for diagnosis.
4. **Read readonly files once**: They don't change.
5. **results.tsv is your memory**: Full experiment history. Re-read when needed.
6. **Short commit messages**: One line, under 80 chars.
