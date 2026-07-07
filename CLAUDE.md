# persistent-memory

A zero-tooling, git-friendly persistent memory layer for coding agents. This repo is both the template and its own first user — the `docs/memory/` files here are live, not just examples.

## Memory
This repo keeps persistent context in `docs/memory/`.
- At the start of a task, read `docs/memory/facts.md` (kept under ~50 lines).
- When you complete a milestone, make a decision, or hit a pitfall, append to `docs/memory/memory-log.jsonl` — format and triggers in `docs/memory/README.md`.
- Search past context with `grep -i "<term>" docs/memory/memory-log.jsonl`.
- Project context belongs in `docs/memory/`, not in your built-in or local memory system. Reserve built-in memory for user preferences only.
