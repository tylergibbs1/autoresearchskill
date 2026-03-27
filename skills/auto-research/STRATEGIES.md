# Research Strategies (When Stuck)

If improvements have stalled, try these in order:

1. **Review results.tsv for patterns** — which categories worked? What failed?
2. **Combine near-misses** — two discarded changes that each almost improved? Try both together.
3. **Try the opposite** — if increasing X hurt, try decreasing X.
4. **Vary magnitude** — small help → try bigger. Small hurt → try milder.
5. **Hunt for unexplored angles** — magic numbers, unused code paths, comments, TODOs.
6. **Read references** — papers, algorithms, or techniques mentioned in the code.
7. **Go radical** — different algorithm, different architecture, completely different approach.
8. **Simplification sweep** — remove components one at a time. Equal metric + less code = win.
9. **Rewind (sparingly)** — `git reset --hard` to earlier known-good commit. Rarely needed.
