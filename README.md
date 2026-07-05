# Persistent Memory

A zero-tooling, git-friendly persistent memory layer for coding agents. Drop it into any repo so Claude, Cursor, Windsurf, or any LLM agent remembers context across sessions, devices, and providers.

## The problem

Every coding session starts blank. You re-explain project quirks, rediscover pitfalls, and re-lay out what you were doing. Agent conversation history doesn't travel between devices or between different agents.

Common solutions to this problem tend to over-engineer it: auto-loading every past entry into context (which burns tokens on irrelevant noise), or building bespoke database-backed memory servers that add setup overhead, sync complexity, and a dependency to maintain. These approaches work against portable, repo-local memory — the very thing that lets context follow your code across devices, agents, and time.

## The solution

Two plain-text files in your repo:

- **`docs/memory/facts.md`** — curated durable truths that stay relevant (max ~50 lines)
- **`docs/memory/memory-log.jsonl`** — append-only chronological log of what happened

Any agent can read and write them with basic shell commands. No database, no API, no special tooling.

## Two-minute setup

```
# 1. Copy the template into your project
cp -r docs/memory/ /path/to/your/project/docs/memory/

# 2. Add the integration to your CLAUDE.md / AGENTS.md / .cursorrules
echo "## Memory" >> CLAUDE.md
echo 'For saving context, logging sessions, and managing project memory, see `docs/memory/README.md`.' >> CLAUDE.md

# 3. Commit
git add docs/memory/ && git commit -m "add persistent memory layer"
```

That's it. Your first session can write to `memory-log.jsonl` and the next session reads it back.

## How it works

One file for what's **always true** (facts.md), one file for what **happened** (memory-log.jsonl). The agent reads them on demand — they're never auto-loaded into context, so there's zero token bloat.

### facts.md

```
- The build tool requires Node 20+
- Test runner --watch doesn't pick up new files without restart
- Prefer `useReducer` over `useState` for form state
```

Kept under ~50 lines. When it grows, the oldest entries get archived to memory-log.jsonl.

### memory-log.jsonl

```
{"date":"2026-07-04","type":"completion","summary":"Task 1: scaffold + vec math done"}
{"date":"2026-07-04","type":"pitfall","summary":"Test runner --watch doesn't pick up new files"}
{"date":"2026-07-04","type":"decision","summary":"ADR-0003: Chose SQLite over Postgres for local dev"}
{"date":"2026-07-04","type":"log","summary":"Session ended. Next: intersection splitting logic."}
```

Append-only, so git merges cleanly across devices. Each line has a `type` (completion, decision, pitfall, fact-archive, log) and a grep-able `summary`.

## Commands cheat sheet

| What | Command |
|------|---------|
| Read facts | `cat docs/memory/facts.md` |
| Add a fact | `echo "- fact goes here" >> docs/memory/facts.md` |
| Recent log | `tail -5 docs/memory/memory-log.jsonl \| sed 's/.*"date":"\([^"]*\)","summary":"\([^"]*\)".*/\1 — \2/'` |
| Search log | `grep -i "term" docs/memory/memory-log.jsonl \| sed 's/.*"date":"\([^"]*\)","summary":"\([^"]*\)".*/\1 — \2/'` |
| Add log entry | `echo '{"date":"$(date +%F)","type":"pitfall","summary":"..."}' >> docs/memory/memory-log.jsonl` |
| Filter by type | `grep '"decision"' docs/memory/memory-log.jsonl` |
| Compact facts | Archive oldest entries to log as `type: "fact-archive"`, remove from facts.md |

## Why JSONL over SQLite

| Concern | JSONL | SQLite | Plain .md |
|---------|-------|--------|-----------|
| Merge across devices | Clean (append-only, no conflicts) | Binary merge conflicts | Conflict-prone if edited in place; safe if append-only |
| Agent-agnostic | grep, tail, echo — works with anything | Needs sqlite3 CLI or a script | cat, grep — works with anything |
| Readable in PRs | Yes — plain text diffs | No — binary blob | Yes |
| Zero tooling | Yes | No | Yes |
| Token cost when idle | Zero (queried on demand) | Zero (queried on demand) | **High if auto-loaded** — agents commonly auto-inject .md into context |
| Structured queries | grep + jq for basic filtering | SQL | grep only — no structured fields |
| Scale limit | ~10K entries | Unlimited | ~1K lines before grep slows |
| Fields | Structured (type, date, summary) | Fully structured | None — plain prose |

JSONL wins for repo-local memory because the deciding factor is **git merges**: two devices committing simultaneously merge cleanly. SQLite's binary format can't. Plain .md files conflict when edited in place and burn tokens if auto-loaded — the worst of both.

## Who is this for

- Anyone using Claude Code, Claude Projects, Cursor, Windsurf, or any LLM coding agent
- Anyone who switches between devices and wants context to follow
- Anyone tired of repeating themselves every session

## What it's not

- Not a replacement for conversation history (that's your agent's job)
- Not a database for structured data (use SQLite for that)
- Not a knowledge base or wiki (use `docs/` for that)

It's just enough memory to avoid starting from zero every session.