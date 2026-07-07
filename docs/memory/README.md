# Memory — repo-local persistent context

A simple, git-friendly system for keeping project context across sessions, devices, and agents. No tooling required — just `grep`, `tail`, and `echo`.

## Why

Coding agents (Claude, Cursor, Windsurf, etc.) start each session with a blank context. Without a memory layer, you repeat yourself — re-explaining project quirks, rediscovering pitfalls, re-laying out the current state. This system stores durable context as plain files in your repo so any agent, on any device, can pick up where you left off.

## File structure

```
docs/memory/
  README.md         ← This file. The workflow reference.
  facts.md          ← Curated durable truths (~50 lines max)
  memory-log.jsonl  ← Append-only chronological log
```

## What goes where

| File | Purpose | Format | Size limit |
|------|---------|--------|------------|
| `facts.md` | Truths that cost time to rediscover. Tech stack quirks, hard-won pitfalls, project-wide conventions. | Markdown list | ~50 lines |
| `memory-log.jsonl` | A chronological record of what happened. Completions, decisions, pitfalls, session notes. Each line is one event. | JSONL (one JSON object per line) | Unlimited |

Both files are plain text in git — zero tooling overhead, zero merge conflicts (the JSONL is append-only).

## When to write

Any of these triggers means something should be saved:

- **A milestone finishes** — task complete, phase done, spec approved
- **A decision is made** — architecture settled, direction chosen
- **A pitfall is discovered** — something that wasted time, so it's not repeated
- **"Save memory" / "save context" / "log session"** — the user explicitly asks
- **Context switch** — switching to a different task or ending a session
- **A durable fact is confirmed** — something that should always be true about the project

## Format

No special tooling: read with `cat`, search with `grep`, append with `echo >>`.

**`facts.md`** is a plain markdown list — one line per fact.

**`memory-log.jsonl`** is one JSON object per line, three fields:

```
{"date":"2026-07-04","type":"pitfall","summary":"Test runner --watch does not pick up new files without restart"}
```

`type` values and when to use them:

| Type | When |
|------|------|
| `completion` | A task, milestone, or phase finished |
| `decision` | A significant choice was made |
| `pitfall` | Something that wasted time — warn future sessions |
| `fact-archive` | A fact rotated out of `facts.md` to keep it short |
| `log` | A general session note that doesn't fit above |

The `summary` is the search surface — `grep -i "<term>"` finds past context, `grep '"pitfall"'` filters by type.

**Appending safely:** if the file ever loses its trailing newline (some editors strip it), the next `echo >>` glues its JSON onto the previous line and silently corrupts the JSONL. If in doubt, repair first:

```
tail -c 1 docs/memory/memory-log.jsonl | read -r _ || echo >> docs/memory/memory-log.jsonl
```

**Compacting facts:** when `facts.md` approaches ~50 lines, move the oldest or least-relevant entries to the log as `type: "fact-archive"` entries, then delete them from `facts.md`.

## Integration with your agent config file

Add this section to your project's `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, or equivalent:

```markdown
## Memory
This repo keeps persistent context in `docs/memory/`.
- At the start of a task, read `docs/memory/facts.md` (kept under ~50 lines).
- When you complete a milestone, make a decision, or hit a pitfall, append to `docs/memory/memory-log.jsonl` — format and triggers in `docs/memory/README.md`.
- Search past context with `grep -i "<term>" docs/memory/memory-log.jsonl`.
- Project context belongs in `docs/memory/`, not in your built-in or local memory system. Reserve built-in memory for user preferences only.
```

Each line earns its place:

- **Reading facts at task start** is what makes pitfalls preventive rather than archaeological — an agent that only reads on request will rediscover the pitfall the hard way first. The ~50-line cap on facts.md exists precisely so this read stays cheap.
- **Inlining the write-triggers** means the agent saves proactively, without needing to open this README first or wait for you to say a magic phrase. (Explicit requests — "save memory", "log session", "has this come up before" — still work too.)
- **The precedence rule** matters for agents with their own memory systems (e.g., Claude Code's local memory directory). Without it, project context gets split between the repo and a machine-local store — and the local half doesn't survive a reformat or follow a clone.

### With ADRs (optional)

If your project uses Architecture Decision Records (ADRs) in `docs/adr/`, cross-reference them in the log:

```
echo '{"date":"2026-07-04","type":"decision","summary":"ADR-0003: Chose SQLite over Postgres for local dev. See docs/adr/0003-sqlite-for-dev.md"}' >> docs/memory/memory-log.jsonl
```

This way, `grep decision` in the log lists every decision ever made and where to find the details, without loading every ADR into context.

## Do's

- **Save early, save often.** A one-line log entry costs nothing. A rediscovered pitfall costs time.
- **Be specific in summaries.** "Fixed a bug" is useless. "Fixed road-split bug where collinear segments produced zero-width lots" is gold.
- **Use grep-friendly language.** Future you will search. Use concrete terms: feature names, function names, error messages.
- **Rotate facts when the file grows.** Keep `facts.md` lean. Archive oldest entries to the log so the current facts are always relevant.
- **Log session endings clearly.** `type: "log"` with what was accomplished, what's next, and any active auths.

## Don'ts

- **Don't save everything.** Not every commit, not every test run, not every trivial fix. Reserve for things worth rediscovering.
- **Don't include secrets.** No API keys, passwords, tokens. These files go in git.
- **Don't append to `facts.md` when `memory-log.jsonl` is the right place.** Durable truths only in facts. Events go in the log.
- **Don't let `facts.md` grow past ~50 lines.** Archive or prune. A bloated facts file defeats its purpose.
- **Don't edit `memory-log.jsonl` in place.** It's append-only. If something is wrong, append a correction entry rather than rewriting history.