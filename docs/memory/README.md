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

## Commands (shell, no tooling required)

### Read facts
```
cat docs/memory/facts.md
```

### Add a fact
```
echo "- The build tool requires Node 20+" >> docs/memory/facts.md
```

### Read recent log entries
```
tail -5 docs/memory/memory-log.jsonl | sed 's/.*"date":"\([^"]*\)","summary":"\([^"]*\)".*/\1 — \2/'
```

### Search the log
```
grep -i "<search-term>" docs/memory/memory-log.jsonl | sed 's/.*"date":"\([^"]*\)","summary":"\([^"]*\)".*/\1 — \2/'
```

Log entries with a `summary` field are grep-able — search for feature names, error messages, people, or technologies.

### Add a log entry
```
echo '{"date":"2026-07-04","type":"pitfall","summary":"Test runner --watch doesn't pick up new files without restart"}' >> docs/memory/memory-log.jsonl
```

**`type` field** values and when to use them:

| Type | When |
|------|------|
| `completion` | A task, milestone, or phase finished |
| `decision` | A significant choice was made |
| `pitfall` | Something that wasted time — warn future sessions |
| `fact-archive` | A fact rotated out of `facts.md` to keep it short |
| `log` | A general session note that doesn't fit above |

All types are grep-able: `grep '"pitfall"' docs/memory/memory-log.jsonl` to see only pitfalls.

### Compact facts (rotate old entries to the log)

When `facts.md` hits ~50 lines, move the oldest/least-relevant entries to the log as `type: "fact-archive"`:

```
# Archive a fact
echo '{"date":"2026-07-04","type":"fact-archive","summary":"Archived from facts.md: Node 20+ requirement"}' >> docs/memory/memory-log.jsonl
# Then remove that line from facts.md
```

## Integration with your agent config file

Add this section to your project's `CLAUDE.md`, `AGENTS.md`, `.cursorrules`, or equivalent:

```markdown
## Memory
For saving context, logging sessions, and managing project memory, see `docs/memory/README.md`.
```

When you say "save memory", "save context", "log session", "remember this", or "has this come up before" — the agent reads this file and knows exactly what to do.

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